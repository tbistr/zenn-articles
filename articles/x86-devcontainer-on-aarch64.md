---
title: "Apple siliconなmacでx86なdevcontainerを使いたい"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "devcontainer", "mac", "arm", "docker"]
published: true
---

## 概要

Macでもdevcontaienrを使って環境構築しようと思いましたが、x86のイメージを開くとVSCodeがスタックしてしまう問題に遭遇しました。

解決のためには、`devcontainer.json`の設定に以下のパラメータを追加します。

```json
"customizations": {
    "vscode": {
        "settings": {
            "extensions.verifySignature": false
        },
    }
}
```

## x86イメージを使う

M3 MacはArmアーキテクチャのSoCを搭載しているため、コンテナを作成するとArmに対応するイメージが使われます。
一方、Rosettaというエミュレータを使うことで、x86のイメージをそのまま実行もできます。
ベースイメージを`FROM --platform=linux/amd64 baseimage`と指定したり、コンテナ起動時に`docker run --platform=linux/amd64`とするとx86のイメージが使用されます。
この時点で勝手にRosseta経由の実行になるようです。

## トラブルの内容

この機能があることを知って、VSCodeのdevcontainerでx86コンテナを指定してみたところ、VSCodeがスタックしてしまいました。
画像のようにコンテナ自体は起動しており、ターミナルも使えるのですが、拡張機能によるシンタックスハイライトやフォーマットは動作していません。
また、左下のステータスバーは「リモートを開いています」から変化しません。

![スタックの様子](https://storage.googleapis.com/zenn-user-upload/76077545d1a9-20240331.png)

## 原因調査

拡張機能がロードされていないので、おそらくその辺かなと思いつつ、原因を調査しました。
事前調査として「aarch64 devcontainer stuck」や「vscode extension loading stuck」のようなキーワードで検索しましたが、クリティカルな情報は見つかりませんでした。

コンテナは起動していてターミナルが使えるので、topやpsコマンドでプロセスを調査してみました。

:::details top

```sh
$ top

top - 14:46:45 up  2:49,  0 users,  load average: 1.59, 1.37, 2.51
Tasks:  18 total,   1 running,  17 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.4 us,  7.8 sy,  0.0 ni, 86.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   7841.3 total,   1264.5 free,   2240.8 used,   4336.0 buff/cache
MiB Swap:   1024.0 total,   1021.0 free,      3.0 used.   5393.0 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  420 vscode    20   0  261.5g  72640  24448 S 100.0   0.9   1:52.31 vsce-sign
    1 root      20   0  287440   4452   1920 S   0.0   0.1   0:00.02 sh
   21 root      20   0  287424   2816   1920 S   0.0   0.0   0:00.00 sh
   27 root      20   0  287400   2560   1792 S   0.0   0.0   0:00.00 sh
   28 vscode    20   0  287440   2944   1920 S   0.0   0.0   0:00.02 sh
   51 root      20   0  287436   2816   1792 S   0.0   0.0   0:00.01 sh
  117 vscode    20   0  287440   2944   1920 S   0.0   0.0   0:00.01 sh
  140 vscode    20   0  929632  64144  35584 S   0.0   0.8   0:00.30 node
  236 vscode    20   0  287444   2944   1920 S   0.0   0.0   0:00.02 sh
  248 vscode    20   0 1253760 123252  38784 S   0.0   1.5   0:02.34 node
  275 vscode    20   0  895760  64716  34816 S   0.0   0.8   0:00.54 node
  302 vscode    20   0  892504  60788  34816 S   0.0   0.8   0:00.28 node
  316 vscode    20   0 1016236  70584  35968 S   0.0   0.9   0:00.45 node
  349 vscode    20   0  953084  84844  36224 S   0.0   1.1   0:00.65 node
  366 vscode    20   0  949772  92072  36096 S   0.0   1.1   0:01.33 node
  378 vscode    20   0  298164  11264   3840 S   0.0   0.1   0:00.07 bash
  937 root      20   0  292200   4832   2304 S   0.0   0.1   0:00.00 sleep
  939 vscode    20   0  297124   7084   3712 R   0.0   0.1   0:00.01 top
```

:::

:::details ps

```sh
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0 287440  4452 ?        Ss   14:44   0:00 /run/rosetta/rosetta /bin/sh /bin/sh -c echo Container started trap "exit 0" 15  exec "$@" while sleep 1 & wait $!; do :; done -
root        21  0.0  0.0 287424  2816 ?        Ss   14:44   0:00 /run/rosetta/rosetta /bin/sh /bin/sh -c echo "New container started. Keep-alive process started." ; export VSCODE_REMOTE_CONTAINERS_SESSION=97f5e20f-3b1e-45d8-a6c4-288e4413a7a41711896287932 ; /bin/sh
root        27  0.0  0.0 287400  2560 ?        S    14:44   0:00 /run/rosetta/rosetta /bin/sh /bin/sh
vscode      28  0.0  0.0 287440  2944 ?        Ss   14:44   0:00 /run/rosetta/rosetta /bin/sh /bin/sh
root        51  0.0  0.0 287436  2816 ?        Ss   14:44   0:00 /run/rosetta/rosetta /bin/sh /bin/sh
vscode     117  0.0  0.0 287440  2944 ?        Ss   14:44   0:00 /run/rosetta/rosetta /bin/sh /bin/sh
vscode     140  0.1  0.7 929632 64144 ?        Sl   14:44   0:00 /run/rosetta/rosetta /home/vscode/.vscode-server/bin/863d2581ecda6849923a2118d93a088b0745d9d6/node /home/vscode/.vscode-server/bin/863d2581ecda6849923a2118d93a088b0745d9d6/node --no-opt -r /proc/.reset /tmp/vscode-remote-containers-server-68a2f162-525f-4b12-9ef8-991c4f4c1e47.js
vscode     236  0.0  0.0 287444  2944 ?        Ss   14:44   0:00 /run/rosetta/rosetta /bin/sh sh /home/vscode/.vscode-server/bin/863d2581ecda6849923a2118d93a088b0745d9d6/bin/code-server --log debug --force-disable-user-env --server-data-dir /home/vscode/.vscode-server --use-host-proxy --telemetry-level all --accept-server-license-terms --host 127.0.0.1 --port 0 --connection-token-file /home/vscode/.vscode-server/data/Machine/.connection-token-863d2581ecda6849923a2118d93a088b0745d9d6 --extensions-download-dir /home/vscode/.vscode-server/extensionsCache --start-server --disable-websocket-compression --skip-requirements-check
vscode     248  1.0  1.5 1253760 123092 ?      Sl   14:44   0:02 /run/rosetta/rosetta /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/node /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/node --no-opt -r /proc/.reset /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/out/server-main.js --log debug --force-disable-user-env --server-data-dir /home/vscode/.vscode-server --use-host-proxy --telemetry-level all --accept-server-license-terms --host 127.0.0.1 --port 0 --connection-token-file /home/vscode/.vscode-server/data/Machine/.connection-token-863d2581ecda6849923a2118d93a088b0745d9d6 --extensions-download-dir /home/vscode/.vscode-server/extensionsCache --start-server --disable-websocket-compression --skip-requirements-check
vscode     275  0.2  0.8 895760 65484 ?        Ssl  14:44   0:00 /run/rosetta/rosetta /home/vscode/.vscode-server/bin/863d2581ecda6849923a2118d93a088b0745d9d6/node /home/vscode/.vscode-server/bin/863d2581ecda6849923a2118d93a088b0745d9d6/node --no-opt -r /proc/.reset -e  ????const net = require('net'); ????const fs = require('fs'); ????process.stdin.pause(); ????const client = net.createConnection({ host: '127.0.0.1', port: 37715 }, () => { ?????console.error('Connection established'); ?????client.pipe(process.stdout); ?????process.stdin.pipe(client); ????}); ????client.on('close', function (hadError) { ?????console.error(hadError ? 'Remote close with error' : 'Remote close'); ?????process.exit(hadError ? 1 : 0); ????}); ????client.on('error', function (err) { ?????process.stderr.write(err && (err.stack || err.message) || String(err)); ????}); ????process.stdin.on('close', function (hadError) { ?????console.error(hadError ? 'Remote stdin close with error' : 'Remote stdin close'); ?????process.exit(hadError ? 1 : 0); ????}); ????process.on('uncaughtException', function (err) { ?????fs.writeSync(process.stderr.fd, `Uncaught Exception: ${String(err && (err.stack || err.message) || err)}\n`); ????}); ???
vscode     302  0.1  0.7 892524 61044 ?        Ssl  14:44   0:00 /run/rosetta/rosetta /home/vscode/.vscode-server/bin/863d2581ecda6849923a2118d93a088b0745d9d6/node /home/vscode/.vscode-server/bin/863d2581ecda6849923a2118d93a088b0745d9d6/node --no-opt -r /proc/.reset -e  ????const net = require('net'); ????const fs = require('fs'); ????process.stdin.pause(); ????const client = net.createConnection({ host: '127.0.0.1', port: 37715 }, () => { ?????console.error('Connection established'); ?????client.pipe(process.stdout); ?????process.stdin.pipe(client); ????}); ????client.on('close', function (hadError) { ?????console.error(hadError ? 'Remote close with error' : 'Remote close'); ?????process.exit(hadError ? 1 : 0); ????}); ????client.on('error', function (err) { ?????process.stderr.write(err && (err.stack || err.message) || String(err)); ????}); ????process.stdin.on('close', function (hadError) { ?????console.error(hadError ? 'Remote stdin close with error' : 'Remote stdin close'); ?????process.exit(hadError ? 1 : 0); ????}); ????process.on('uncaughtException', function (err) { ?????fs.writeSync(process.stderr.fd, `Uncaught Exception: ${String(err && (err.stack || err.message) || err)}\n`); ????}); ???
vscode     316  0.2  0.8 1147348 70712 ?       Sl   14:44   0:00 /run/rosetta/rosetta /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/node /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/node --no-opt -r /proc/.reset /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/out/bootstrap-fork --type=fileWatcher
vscode     349  0.2  1.0 953196 85996 ?        Sl   14:44   0:00 /run/rosetta/rosetta /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/node /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/node --no-opt -r /proc/.reset --dns-result-order=ipv4first /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/out/bootstrap-fork --type=extensionHost --transformURIs --useHostProxy=true
vscode     366  0.6  1.1 949820 93060 ?        Sl   14:44   0:01 /run/rosetta/rosetta /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/node /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/node --no-opt -r /proc/.reset /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/out/bootstrap-fork --type=ptyHost --logsPath /home/vscode/.vscode-server/data/logs/20240331T144451
vscode     378  0.0  0.1 298164 11264 pts/0    Ss   14:44   0:00 /run/rosetta/rosetta /bin/bash /bin/bash --init-file /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/out/vs/workbench/contrib/terminal/browser/media/shellIntegration-bash.sh
vscode     420 99.9  0.9 274208228 72640 ?     Sl   14:44   3:45 /run/rosetta/rosetta /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/node_modules/@vscode/vsce-sign/bin/vsce-sign /vscode/vscode-server/bin/linux-x64/863d2581ecda6849923a2118d93a088b0745d9d6/node_modules/@vscode/vsce-sign/bin/vsce-sign verify --package /home/vscode/.vscode-server/extensionsCache/ms-ceintl.vscode-language-pack-ja-1.87.2024031309 --signaturearchive /home/vscode/.vscode-server/extensionsCache/ms-ceintl.vscode-language-pack-ja-1.87.2024031309.sigzip
root      1141  0.0  0.0 292200  4828 ?        S    14:48   0:00 /run/rosetta/rosetta /bin/sleep sleep 1
vscode    1143  0.0  0.0 296724  6504 pts/0    R+   14:48   0:00 /bin/ps ps aux
```

:::

topコマンドによると、`vsce-sign`というプロセスがCPUを100％使っています。
また、psコマンドによると`vsce-sign`はやはりRosetta経由で実行されているようです。
今回の問題は、おそらくRosettaがx86の一部命令（AVXとか？）をサポートしていないことが原因だと思われます。

名前的に`vsce-sign`は拡張機能の署名を検証するためのプロセスだと予測しました。
そこで、「vscode devcontainer extension signature check stuck」で検索してみると、[こちらのissue](https://github.com/microsoft/vscode/issues/174632)が見つかりました。
ずらっとコメントが並んでいますが、[こちらのコメント](https://github.com/microsoft/vscode/issues/174632#issuecomment-1608114637)で拡張機能の署名検証をスキップする設定が紹介されていました。
`"extensions.verifySignature": false`らしいです。

## 解決策

調査によって得られた設定を検証してみましたが、単純にユーザー設定に追加するだけでは解決しませんでした。
スレッドにも成功した人と失敗している人がいて、手元でも色々試した結果`devcontainer.json`に設定を追加することで解決しました。

```json
"customizations": {
    "vscode": {
        "settings": {
            "extensions.verifySignature": false
        },
    }
}
```

このように`devcontainer.json`に設定を追加することで、拡張機能の署名検証をスキップできました。
ワークスペースの`.vscode/settings.json`や、ユーザー設定ではdevcontaienr環境には影響しないようです。
これでスタックする問題も解決して、無事にx86のdevcontainerを使うことができました。
あくまで予想ですが、拡張機能の署名検証を始めるタイミングと設定を読み込むタイミングの関係が原因と思われます。

### 再起動時

どうやら、拡張機能の検証はコンテナを再起動する場合（拡張機能がコンテナにインストールされている場合）スキップされるようです。
**そのため、初回起動時のみ`devcontainer.json`に設定を書いておけば、開いた後には消しても問題ありませんでした。**
これによって、チームメンバーがx86環境な場合でも、この環境特有な設定をGit管理に載せないようにできます。

## まとめ

使えるようになったコンテナは意外と快適で、ヘビーな作業をしなければRosetta経由でも問題なさそうです。

調査結果は[GitHubのissueにもフィードバック](https://github.com/microsoft/vscode/issues/174632#issuecomment-2024954684)しました。
