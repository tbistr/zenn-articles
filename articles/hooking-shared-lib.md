---
title: "LD_PRELOADを駆使してロードする共有ライブラリを動的に変える(dlopen対応)"
emoji: "🎣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["linux"]
published: true
publication_name: "kurusugawa"
---

## 概要

- 共有ライブラリの関数呼び出しをフックする方法について解説
- `LD_PRELOAD` を使用した共有ライブラリのフック方法を紹介
- `dlopen` と `dlsym` を使用してロードされた関数をフックする方法
- 自身をロードして公開するテクニックにより、`dlopen` もフック可能

https://github.com/tbistr/so-intercept-hands-on

## `LD_PRELOAD`による共有ライブラリのフック

Linuxでは、`LD_PRELOAD` を用いて共有ライブラリの関数をフックすることが一般的です。
`LD_PRELOAD`に共有ライブラリのパスを指定すると、プログラム起動時に動的リンカが対象のライブラリを優先的にリンクします。
これにより、指定ライブラリ内の関数が同名の既存関数を置き換える形で動作します。

以下のC++プログラムは `math.h` を介して `libm.so` の `pow` 関数と `log` 関数を呼び出します。

```cpp:main.cpp
#include <iostream>
#include <math.h>

int main()
{
    double result;
    // Use variables to avoid compiler optimization
    auto x = 8.0;
    auto y = 2.0;
    result = pow(x, y);
    std::cout << "pow(8.0, 2.0) = " << result << std::endl;

    auto e = 2.718282;
    result = log(e);
    std::cout << "log(e) = " << result << std::endl;
}

```

```bash:プログラムの実行結果
$ g++ -O0 -o main main.cpp -lm && ./main
pow(8.0, 2.0) = 64
log(e) = 1
```

ここで、フックしたい関数(この例では`pow`)と同名、同シグネチャの関数を定義して共有ライブラリを作成します。
このライブラリでは、`pow`に渡される引数を勝手に倍にしてしまう処理を挟んでいます。
また、オリジナルの`pow`を呼び出すために`dlsym`の`RTLD_NEXT`フラグを使用しています。

```cpp:intercept.cpp
#include <dlfcn.h>
#include <iostream>

static double (*original_pow)(double, double) = nullptr;

extern "C" double pow(double x, double y)
{
    // Load original pow() if not loaded yet
    if (!original_pow)
    {
        original_pow = reinterpret_cast<double (*)(double, double)>(dlsym(RTLD_NEXT, "pow"));
        if (!original_pow)
        {
            std::cout << "Error loading original pow function." << std::endl;
            std::exit(EXIT_FAILURE);
        }
    }

    // Do some interception
    x *= 2;
    y *= 2;

    std::cout << "\033[31m";
    std::cout << "Indercepted! Args are doubled: (" << x << ", " << y << ")" << std::endl;
    std::cout << "\033[0m";

    // Call original pow()
    return original_pow(x, y);
}
```

この共有ライブラリをビルド、`LD_PRELOAD`環墧変数に指定してプログラムを実行すると、`pow`関数の引数が2倍にされていることが確認できます。

```bash:処理をフックしたプログラムの実行結果
$ g++ -O0 -o main main.cpp -lm
$ g++ -O0 -shared -fPIC -o intercept.so intercept.cpp -ldl
$ LD_PRELOAD=./intercept.so ./main
Indercepted! Args are doubled: (16, 4)
pow(8.0, 2.0) = 65536
log(e) = 1
```

これが、共有ライブラリのフックの基本的な方法です。

## `dlopen`による共有ライブラリのロード

先述のプログラム(`main.cpp`)では、実行環境の動的リンカがプログラム起動時に`libm.so`をリンクしていました。
一方で`dlopen`関数を使用することで、プログラム実行中に任意の共有ライブラリを動的にロードできます。

以下のプログラムでは、`dlopen`を使用して`libm.so`をロードし、`dlsym`で`pow`と`log`のアドレスを取得して呼び出しています。

```cpp:main_via_dlopen.cpp
#include <iostream>
#include <stdlib.h> // for exit()
#include <dlfcn.h>  // for dlopen(), dlsym()
// we don't need math.h anymore

int main()
{
    char *error;

    // load libm.so dynamically
    auto handle = dlopen("libm.so", RTLD_LAZY);
    if (!handle)
    {
        fprintf(stderr, "%s\n", dlerror());
        exit(EXIT_FAILURE);
    }
    dlerror();

    // get pow() address
    auto dl_pow = (double (*)(double, double))dlsym(handle, "pow");
    if ((error = dlerror()) != NULL)
    {
        fprintf(stderr, "%s\n", error);
        exit(EXIT_FAILURE);
    }

    // get log() address
    auto dl_log = (double (*)(double))dlsym(handle, "log");
    if ((error = dlerror()) != NULL)
    {
        fprintf(stderr, "%s\n", error);
        exit(EXIT_FAILURE);
    }

    // do same thing as main.cpp
    double result;
    auto x = 8.0;
    auto y = 2.0;
    result = dl_pow(x, y);
    std::cout << "pow(8.0, 2.0) = " << result << std::endl;

    auto e = 2.718282;
    result = dl_log(e);
    std::cout << "log(e) = " << result << std::endl;

    // unload libm.so
    dlclose(handle);
    return 0;
}
```

このプログラムの実行結果は先述の`main.cpp`と同じです。
どちらの方法でも同じ関数が呼び出されていることが確認できます。

```bash:dlopenを使ったプログラムの実行結果
$ g++ -O0 -o main_via_dlopen main_via_dlopen.cpp -ldl
$ ./main_via_dlopen
pow(8.0, 2.0) = 64
log(e) = 1
```

さて、LD_PRELOADを使ってmain_via_dlopenの中で使われている関数をフックできるでしょうか？

残念ながら、`LD_PRELOAD`によるフックは`dlopen`でロードしたライブラリには効果がありません。
これは、`LD_PRELOAD`があくまでプログラム起動時のリンクに対して働くためです。

```bash:LD_PRELOADを使ってフックしてみる
$ LD_PRELOAD=./intercept.so ./main_via_dlopen
pow(8.0, 2.0) = 64
log(e) = 1
```

## `dlopen`でロードしたライブラリをフックする

なんとかして、`dlsym`で検索される対象に自作の`pow`関数を含める必要があります。
そのための方法として、`dlopen`をフックして自作の共有ライブラリをロードする方法があります。

```cpp:intercept_dl.cpp
#include <dlfcn.h>
#include <string>
#include <iostream>

static void *(*original_dlopen)(const char *filename, int flags) = nullptr;

extern "C" void *dlopen(const char *filename, int flags)
{
    if (!original_dlopen)
    {
        original_dlopen = reinterpret_cast<void *(*)(const char *, int)>(dlsym(RTLD_NEXT, "dlopen"));
        if (!original_dlopen)
        {
            std::cout << "Error loading original dlopen function." << std::endl;
            std::exit(EXIT_FAILURE);
        }
    }

    if (std::string(filename) == "libm.so")
    {
        // Set dummy variable, it has static address.
        static int dummy = 0xdeedbeef;
        Dl_info info;
        // Find .so info which contains dummy variable address (=ownself).
        if (dladdr(&dummy, &info))
        {
            // Load ownself instead of libm.so
            // This program have loaded libm.so implicitly (see ldd ./intercept_dl.so).
            // Thus we can expose symbols of original libm.so.
            // If you want to hook library which is not loaded yet, you should load it explicitly with RTLD_GLOBAL.
            // ex. original_dlopen("libm.so", RTLD_LAZY | RTLD_GLOBAL);
            return original_dlopen(info.dli_fname, flags);
        }
    }
    return original_dlopen(filename, flags);
}
```

注目すべきは`dladdr`関数の利用です。
staticに定義された変数`dummy`のアドレスを検索することで、自分自身のパスを取得しています。
これを`pow`を公開している`intercept.cpp`と一緒に共有ライブラリとしてビルド、`main_via_dlopen.cpp`と一緒に実行すると、`pow`関数がフックされていることが確認できます。

```bash:intercept_dl.soを使ってフックしてみる
$ g++ -O0 -shared -fPIC -o intercept_dl.so intercept.cpp intercept_dl.cpp -ldl
$ LD_PRELOAD=./intercept_dl.so ./main_via_dlopen
Indercepted! Args are doubled: (16, 4)
pow(8.0, 2.0) = 65536
log(e) = 1
```

見事に`dlopen`でロードしたライブラリもフックすることに成功しました。

## 非フック対象関数の実行

さて、`dlopen`をフックすることで`libm.so`の代わりに`intercept_dl.so`をロードできました。
しかし、`libm.so`の他の関数はどうなるでしょうか？
今回のプログラムでは、`log`関数をフックしていませんが、`main_via_dlopen.cpp`から呼び出しています。
`intercept_dl.so`では`log`関数を定義していないため、`dlsym`の結果`libm.so`の`log`関数が呼び出されて欲しいところです。

実際のところ実行結果を見て分かるように、ちゃんと`libm.so`の`log`関数が呼び出されています。
これは`intercept_dl.so`が暗黙に`libm.so`をロード、ロードしたシンボルを外部に公開しているためです。
`ldd`コマンドで`intercept_dl.so`を調べると`libm.so`に依存していることが分かります。

```bash:intercept_dl.soの依存関係
$ ldd ./intercept_dl.so
linux-vdso.so.1 (0x0000ffff9d861000)
libdl.so.2 => /lib/aarch64-linux-gnu/libdl.so.2 (0x0000ffff9d80a000)
libstdc++.so.6 => /usr/lib/aarch64-linux-gnu/libstdc++.so.6 (0x0000ffff9d632000)
libgcc_s.so.1 => /lib/aarch64-linux-gnu/libgcc_s.so.1 (0x0000ffff9d60e000)
libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000ffff9d49a000)
/lib/ld-linux-aarch64.so.1 (0x0000ffff9d831000)
libm.so.6 => /lib/aarch64-linux-gnu/libm.so.6 (0x0000ffff9d3ef000)
```

これは`iostream`などのライブラリが`libm.so`をリンクしているためです。
`intercept_dl.so`のロード時に`libm.so`も自動的にロードされ、`intercept_dl.so`の中で見つからなかったシンボルは`libm.so`から探されるようになります。

しかし、`libm.so`がリンクされているのはたまたまであり、他のライブラリでは同じようなことが期待できません。
従って、明示的に非フック対象を含むライブラリをロード、公開しておくことが重要です。
その場合、`dlopen`のフラグに`RTLD_GLOBAL`を指定することでロードしたライブラリのシンボルを他のライブラリに公開できます。
(`dlopen("libm.so", RTLD_LAZY | RTLD_GLOBAL);`)

## まとめ

- `LD_PRELOAD`はプログラム起動時のリンクに対して働く
- `dlopen`でロードしたライブラリは`LD_PRELOAD`によるフックの対象外
- `dlopen`をフックして自分自身をロードすることで、`dlsym`で検索される対象に自作の関数を含めることができる
- 非フック対象関数を実行するためには、明示的に非フック対象を含むライブラリをロード、公開しておく
- `RTLD_GLOBAL`フラグを指定することでロードしたライブラリのシンボルを他のライブラリに公開できる

今回の例はマルチメディア系のアプリ等、実行環境によって動的にロードするライブラリが変わるような場合に有用です。
実際、このアイデアは[OpenGLのトレースツールである`apitrace`](https://github.com/apitrace/apitrace)のコード中に発見しました。

また、今回の実行プログラムと実行環境は[こちらのリポジトリ](https://github.com/tbistr/so-intercept-hands-on)で試すことができます。
