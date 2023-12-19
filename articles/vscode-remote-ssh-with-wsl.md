---
title: "VS CodeのRemote SSHでWSL, Windows両方の設定を使う"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "ssh", "wsl"]
published: true
publication_name: "kurusugawa"
---

この記事では、VS CodeのRemote SSH拡張機能を使って、WSLとWindowsの両方の設定でSSHターゲットに接続する方法を紹介します。

# Remote SSH拡張機能

VS CodeのRemote SSH拡張機能は、VS CodeからSSH接続先のファイルを編集するための拡張機能です。
詳細と使い方は[VS Codeのドキュメント](https://code.visualstudio.com/docs/remote/ssh)で解説されています。

接続先はSSHのconfigファイルから読み込まれます。
また、使用するSSHクライアント、設定ファイルのパスはVS Codeの設定で指定できます。

```json
  "remote.SSH.path": "path/to/ssh/binary",
  "remote.SSH.configFile": "path/to/ssh/config",
```

デフォルトではWindowsのOpenSSHクライアントとホームディレクトリの`.ssh/config`ファイルが使われます。

# 課題

私はWSLをメインに使っているため、WSLのSSHクライアントと設定ファイルの内容を使って接続したいです。
また、WSLの他にVagrantなどの仮想環境を使っているので、仮想マシン内にWindowsのSSHクライアントと設定ファイルを使って接続をしたいこともあります。

つまり、VS CodeのRemote SSH拡張機能で、**WSLとWindows両方のクライアント、設定を使い分けたい**ということです。

しかし、WSL側のconfigファイル、SSHクライアントを使った接続はできません。
この問題は[GitHubのissue](https://github.com/microsoft/vscode-remote-release/issues/937)にもなっていて、リリース初期から未解決のままです。
例えば、以下のように無理矢理WSLのSSHクライアントを使うように設定してもうまくいかないことが報告されています。

```json
    "remote.SSH.path": "\\wsl$\Ubuntu-18.04\usr\bin\ssh"
    "remote.SSH.configFile": "\\wsl$\Ubuntu-18.04\home\user\.ssh\config"
```

解決策として[Windowsのbatファイルを使ってWSLのSSHクライアントを呼び出す方法](https://github.com/microsoft/vscode-remote-release/issues/937#issuecomment-578416517)が紹介されています。
ただし、この方法ではWindows側のSSHクライアントの使用を完全にオミットすることになります。
つまり、**Windows側のクライアントとconfigファイルは使えなくなる**ということです。

# 解決

この問題を解決するために、SSHの接続先によって使用するSSHクライアントと設定ファイルを自動的に切り替えるコマンドを作りました。
`wsl-ssh-amb`という名前で[GitHubに公開しています](https://github.com/tbistr/wsl-ssh-amb)。
WSL側SSHクライアント実行の代行者(ambassador)という意味で`wsl-ssh-amb`という名前にしました。

## コマンドの使い方

[リリースページ](https://github.com/tbistr/wsl-ssh-amb/releases/tag/v0.1.0)からビルド済みのバイナリをダウンロードして、任意の場所に保存します。
次にVS CodeのRemote SSH拡張機能の設定で、SSHクライアントのパスを`wsl-ssh-amb`に設定します。

```json
  "remote.SSH.path": "path/to/wsl-ssh-amb",
  "remote.SSH.remoteServerListenOnSocket": true, // おまじない
```

接続先一覧に表示させるため、SSHのconfigファイルに接続先を設定します。
一覧に表示されるのは、Windows側の設定ファイル(デフォルトで`~/.ssh/config`)の内容です。
WSL側のクライアント、設定を使うために、`wsl_`で始まり、WSL側での接続先名が続くような名前をWindows側の設定ファイルに追加します。

例えば、WSL側の設定ファイルが以下のようになっている場合を考えます。

```ssh-config
Host server_ubuntu
  HostName ubuntu.example.com
  Port 2222
  User user
  IdentityFile ~/.ssh/id_rsa
Host server_centos
  HostName centos.example.com
  IdentityFile ~/.ssh/id_rsa_centos
```

ここで、`server_ubuntu`にRemote SSHで接続したい場合、**Windows側の設定ファイルに**以下の項目を追加します。
設定名以外の項目を指定する必要はありません。

```ssh-config
Host wsl_server_ubuntu
```

これにより、Remote SSHの接続先一覧に`wsl_server_ubuntu`が表示されるようになります。
この設定を選択すると、`wsl-ssh-amb`が実行され、WSL側のSSHクライアントと設定ファイルを使って`server_ubuntu`に接続がされます。

私の環境では以下のような表示になり、適切にWSL側のクライアント、設定が使われています。

![接続先一覧](https://storage.googleapis.com/zenn-user-upload/d1f113ff486b-20231217.png)

## 原理説明

そもそも、VS CodeのRemote SSH拡張機能は、SSHのコマンドを直接実行しています。
デバッグの結果、それぞれのOSに特殊な引数は渡していないことがわかりました。
そのため、コマンド内では双方に同じ引数を渡しています。

:::message
ただし`-F`オプション(設定ファイルのパス指定)については、Windowsのパス表現を使っています。
そのためこのコマンドではWSL側には渡さないようにしており、WSL側の設定ファイルはデフォルトの`~/.ssh/config`が使われます。
:::

コマンド内部では、渡された引数をパースしてターゲットの設定を探し、ターゲット名が`wsl_`で始まる場合はWSL側、そうでない場合はWindows側のSSHコマンドを実行しています。

# まとめ

VS CodeのRemote SSH拡張機能を使って、WSLとWindows両方での接続を共存させる方法を紹介しました。
実はちょっと前に書いたコマンドなんですが、記事にするのを忘れていました。

処理内容は簡単で短いので、PowerShellなどで実装したほうが便利かもしれません。
また、SSH-Agentの対応を確認できていないので、そこもチェックしたいです。
