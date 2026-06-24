---
title: "C++のベンチマークライブラリ(ヘッダのみ)を使ってみる"
emoji: "👋"
type: "tech"
topics:
  - "cpp"
  - "linux"
  - "gcc"
published: true
published_at: "2025-02-26 03:57"
---

# 序
今回試してみたのは

- nanobench
- mitata

というヘッダのみの軽量級のものです。mitataはjsでは知ってる人もいるかもしれないけど、C++ではまるで聞かないライブラリなので、その辺を調べてみました。
C++で他に有名なのはGoogle Bnechmarkですが、これはヘッダのみで済まないので除外しています。

https://github.com/martinus/nanobench
https://github.com/evanwashere/mitata

※元は20日ほど前にQiitaに載せた記事ですが、編集不可能になってしまったので(将来的には削除予定)、こちらに引っ越しています。

# 1. 目的
以下のmicro benchmark libraryの使用感を確かめる

- nanobench
- mitata

# 2. 測定対象
適当に用意したコード

```cpp:example.hpp
#pragma once
extern bool write_by_systemcall(const char* path, size_t size) noexcept;
extern bool write_by_stdc(const char* path, size_t size) noexcept;
extern bool write_by_stdcpp(const char* path, size_t size) noexcept;
extern bool write_hello_by_stdc(const char* path, size_t size) noexcept;
extern bool write_hello_by_stdcpp(const char* path, size_t size) noexcept;
```

この実装を共有ライブラリにして測定します。各関数は全て約100MiBのファイル作成ですが、それぞれ以下の特徴があります。

|関数|使用するもの|テキスト/バイナリ|書き込み単位|
|----|----|----|----|
|write_by_systemcall|システムコール直呼び出し|バイナリ|1MiB|
|write_by_stdc|標準Cライブラリ|バイナリ|1MiB|
|write_by_stdcpp|標準C++ライブラリ|バイナリ|1MiB|
|write_hello_by_stdc|標準Cライブラリ|テキスト|1行|
|write_hello_by_stdcpp|標準C++ライブラリ|テキスト|1行|

共有ライブラリにするのは測定対象を結合した状態で最適化させないためです(何を測ってるのか分からなくなるため)。コードの詳細は以下を見てください。

:::details ソースコード(押すと展開)

```cpp:example.cpp
#include <fcntl.h>
#include <unistd.h>
#include <cstdlib>
#include <vector>
#include <cstdio>
#include <iostream>
#include <ios>
#include <fstream>
#include "example.hpp"
const size_t BUFFER_SIZE = 1024 * 1024;
bool write_by_systemcall(const char* path, size_t size) noexcept {
    std::vector<char> buffer(BUFFER_SIZE);
    int fd = open(path, O_CREAT | O_WRONLY | O_TRUNC, 0666);
    if (fd == -1) {
        perror(__PRETTY_FUNCTION__);
        return false;
    }
    while (size > 0) {
        size_t len = buffer.size() > size ? size : buffer.size();
        if (write(fd, buffer.data(), len) != static_cast<ssize_t>(len)) {
            close(fd);
            perror(__PRETTY_FUNCTION__);
            return false;
        }
        size -= len;
    }
    close(fd);
    return true;
}
bool write_by_stdc(const char* path, size_t size) noexcept {
    std::vector<char> buffer(BUFFER_SIZE);
    FILE* fp = fopen(path, "wb");
    if (fp == NULL) {
        perror(__PRETTY_FUNCTION__);
        return false;
    }
    while (size > 0) {
        size_t len = buffer.size() > size ? size : buffer.size();
        if (fwrite(buffer.data(), sizeof(decltype(buffer)::value_type), len, fp) != static_cast<size_t>(len)) {
            fclose(fp);
            perror(__PRETTY_FUNCTION__);
            return false;
        }
        size -= len;
    }
    fclose(fp);
    return true;
}
bool write_by_stdcpp(const char* path, size_t size) noexcept {
    std::vector<char> buffer(BUFFER_SIZE);
    std::ofstream f(path, std::ios::out | std::ios::binary);
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    while (size > 0) {
        size_t len = buffer.size() > size ? size : buffer.size();
        f.write(buffer.data(), len);
        if (! f) {
            std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
            return false;
        }
        size -= len;
    }
    f.flush();
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    return true;
}
bool write_hello_by_stdc(const char* path, size_t size) noexcept {
    FILE* fp = fopen(path, "wt");
    if (fp == NULL) {
        perror(__PRETTY_FUNCTION__);
        return false;
    }
    long l = 0;
    while (size > 0) {
        int len;
        if ((len = fprintf(fp, "hello! %ld\n", l++)) < 0) {
            fclose(fp);
            perror(__PRETTY_FUNCTION__);
            return false;
        }
        size -= size > static_cast<size_t>(len) ? len : size;
    }
    fclose(fp);
    return true;
}
bool write_hello_by_stdcpp(const char* path, size_t size) noexcept {
    std::ofstream f(path, std::ios::out);
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    long l = 0;
    while (size > f.tellp()) {
        f << "hello! " << l++ << "\n";
        if (! f) {
            std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
            return false;
        }
    }
    f.flush();
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    return true;
}
```

```shell-session
$ g++ -shared -fPIC -g -Wall -pedantic -std=c++17 -O3 example.cpp -o libexample.so
```

:::

# 3. nanobenchの使い方

```cpp
#define ANKERL_NANOBENCH_IMPLEMENT
#include "nanobench.h" // https://raw.githubusercontent.com/martinus/nanobench/v4.3.11/src/include/nanobench.h
...
  {
    std::cout << "======================== nanobench ========================" << std::endl;
    ankerl::nanobench::Bench().run("standard C++", [&] {
      assert(write_by_stdcpp("test.bin", 1024 * 1024 * 100));
    });
    ankerl::nanobench::Bench().run("standard C", [&] {
      assert(write_by_stdc("test.bin", 1024 * 1024 * 100));
    });
    ankerl::nanobench::Bench().run("system call", [&] {
      assert(write_by_systemcall("test.bin", 1024 * 1024 * 100));
    });
  }
```

assertで測ってるので最適化してもNDEBUGしないで下さい。
内容的には説明するまでもないので、詳細は以下で。

https://nanobench.ankerl.com/

# 4. mitataの使い方
こちらはまともじゃないです。まず使用するのに以下のincludeが必要です。

```cpp
#include <chrono>
#include <cmath>
#include <cstdint>
```

なぜか必須なのにヘッダに書いていません。またC++20で標準化された指示付き初期化が使用されているのですが、順番通りに初期化されておらず、コードの修正が必要です。

```diff cpp
--- a/mitata.hpp	2025-02-05 14:54:25.926948220 +0900
@@ -217,11 +217,11 @@ namespace mitata {
       for (auto o = 0; o <= poffset; o++) bins[std::round((stats.samples[o] - min) / step)]++;
 
       return {
+        .avg = clamp(0, (u64)std::round((stats.avg - min) / step), size - 1),
+        .peak = *std::max_element(bins.begin(), bins.end()),
+        .outliers = stats.samples.size() - 1 - poffset,
         .min = min, .max = max,
         .step = step, .bins = bins, .steps = steps,
-        .outliers = stats.samples.size() - 1 - poffset,
-        .peak = *std::max_element(bins.begin(), bins.end()),
-        .avg = clamp(0, (u64)std::round((stats.avg - min) / step), size - 1),
       };
     }
 
```

これだけしてようやくコンパイルが通りますが、それでも大量のワーニング付きです。
本題の使い方ですが、以下な感じです。

```cpp
#include <chrono>
#include <cmath>
#include <cstdint>
...
#include "mitata.hpp"  // https://github.com/evanwashere/mitata/raw/refs/tags/v1.0.23/src/mitata.hpp
...
  {
    std::cout << "======================== mitata ========================" << std::endl;
    mitata::runner runner;
    runner.summary([&]() {
      runner.bench("standard C++", []() {assert(write_by_stdcpp("test.bin", 1024 * 1024 * 100));});
      runner.bench("standard C", []() {assert(write_by_stdc("test.bin", 1024 * 1024 * 100));});
      runner.bench("system call", []() {assert(write_by_systemcall("test.bin", 1024 * 1024 * 100));});
    });
    auto stats = runner.run();
  }
...
```

githubのexampleだけでドキュメントはないのでコード参照です。

# 5. 計測結果

RAMディスク上(/run/user/[uid]/)で計測しています。測定用のコード全体は記事の末尾に載せています。

```shell-session
$ LD_LIBRARY_PATH=$(pwd) ./test
======================== mitata ========================
runtime: c++
compiler: gcc
benchmark                   avg (min … max) p75   p99    (min … top 1%)
------------------------------------------- -------------------------------
standard C                    93.81 ms/iter  94.32 ms           █▃         
                      (92.57 ms … 95.24 ms)  94.72 ms ▆▁▆▁▆▁▁▁▁▁██▁▁▁▁▆▁▁▆▆
standard C++                  94.37 ms/iter  95.15 ms                   █  
                      (92.57 ms … 95.73 ms)  95.37 ms █▁▁▁█▁▁██▁█▁█▁▁▁█▁███
system call                   94.11 ms/iter  94.74 ms █                    
                      (92.94 ms … 95.58 ms)  95.42 ms █▁██▁▁▁█▁███▁▁██▁▁▁▁█
summary
  standard C
   1x faster than system call
   1.01x faster than standard C++
======================== nanobench ========================
Warning, results might be unstable:
* CPU frequency scaling enabled: CPU 0 between 1,550.0 and 3,200.0 MHz
* CPU governor is 'schedutil' but should be 'performance'
Recommendations
* Use 'pyperf system tune' before benchmarking. See https://github.com/psf/pyperf
|               ns/op |                op/s |    err% |     total | benchmark
|--------------------:|--------------------:|--------:|----------:|:----------
|       94,465,779.00 |               10.59 |    1.5% |      1.03 | `standard C++`
|       92,495,337.00 |               10.81 |    0.9% |      1.02 | `standard C`
|       89,882,275.00 |               11.13 |    0.8% |      0.99 | `system call`
======================== mitata ========================
runtime: c++
compiler: gcc
benchmark                   avg (min … max) p75   p99    (min … top 1%)
------------------------------------------- -------------------------------
standard C                   725.25 ms/iter 728.91 ms ██ █  ██   ████    ██
                    (712.21 ms … 737.53 ms) 736.93 ms ██▁█▁▁██▁▁▁████▁▁▁▁██
standard C++                    2.82 s/iter    2.84 s                   █ █
                          (2.76 s … 2.86 s)    2.84 s █▁▁█▁▁█▁▁▁█▁▁▁▁████▁█
summary
  standard C
   3.88x faster than standard C++
======================== nanobench ========================
|    2,909,028,911.00 |                0.34 |    0.5% |     31.65 | `standard C++`
|      728,964,797.00 |                1.37 |    0.8% |      8.07 | `standard C`
$ 
```

計測結果自体は予想通りで、どちらも有意な差はありませんでした。
C++版mitataは見た目だけで中身がボロボロだというのがよく分かりました。js版も使う気がなくなった感じです。

計測結果を考察することは目的から外れますが、一応書いておくと

- バイナリアクセスではシステムコール直呼びに匹敵する性能をC/C++ともに見せた
- テキストの小さな処理はC++が4倍程度遅い結果となった

2番目の主な原因は~~streambufの仮想関数呼び出し[std::basic_streambuf::xsputn](https://cpprefjp.github.io/reference/streambuf/basic_streambuf/xsputn.html)辺りで最適化が抑制されてしまうから、だと思います(推測)~~ `tellp()`でseekが発生していたからです(2026年6月24日編集)。
あとコメントしてありますが、今回のコードでは`std::ios_base::sync_with_stdio`の影響はほぼありませんでした。

# 6. まとめ

- mitataは見た目は良いけど中身が悪かった
- nanobenchは見た目は普通だけど中身は堅実だった

※どちらも使えないことはない

# 付録

```sh:test.sh
set -eux
cat >example.hpp <<EOF
#pragma once
extern bool write_by_systemcall(const char* path, size_t size) noexcept;
extern bool write_by_stdc(const char* path, size_t size) noexcept;
extern bool write_by_stdcpp(const char* path, size_t size) noexcept;
extern bool write_hello_by_stdc(const char* path, size_t size) noexcept;
extern bool write_hello_by_stdcpp(const char* path, size_t size) noexcept;
EOF
cat >example.cpp <<EOF
#include <fcntl.h>
#include <unistd.h>
#include <cstdlib>
#include <vector>
#include <cstdio>
#include <iostream>
#include <ios>
#include <fstream>
#include "example.hpp"
const size_t BUFFER_SIZE = 1024 * 1024;
bool write_by_systemcall(const char* path, size_t size) noexcept {
    std::vector<char> buffer(BUFFER_SIZE);
    int fd = open(path, O_CREAT | O_WRONLY | O_TRUNC, 0666);
    if (fd == -1) {
        perror(__PRETTY_FUNCTION__);
        return false;
    }
    while (size > 0) {
        size_t len = buffer.size() > size ? size : buffer.size();
        if (write(fd, buffer.data(), len) != static_cast<ssize_t>(len)) {
            close(fd);
            perror(__PRETTY_FUNCTION__);
            return false;
        }
        size -= len;
    }
    close(fd);
    return true;
}
bool write_by_stdc(const char* path, size_t size) noexcept {
    std::vector<char> buffer(BUFFER_SIZE);
    FILE* fp = fopen(path, "wb");
    if (fp == NULL) {
        perror(__PRETTY_FUNCTION__);
        return false;
    }
    while (size > 0) {
        size_t len = buffer.size() > size ? size : buffer.size();
        if (fwrite(buffer.data(), sizeof(decltype(buffer)::value_type), len, fp) != static_cast<size_t>(len)) {
            fclose(fp);
            perror(__PRETTY_FUNCTION__);
            return false;
        }
        size -= len;
    }
    fclose(fp);
    return true;
}
bool write_by_stdcpp(const char* path, size_t size) noexcept {
    std::vector<char> buffer(BUFFER_SIZE);
    std::ofstream f(path, std::ios::out | std::ios::binary);
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    while (size > 0) {
        size_t len = buffer.size() > size ? size : buffer.size();
        f.write(buffer.data(), len);
        if (! f) {
            std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
            return false;
        }
        size -= len;
    }
    f.flush();
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    return true;
}
bool write_hello_by_stdc(const char* path, size_t size) noexcept {
    FILE* fp = fopen(path, "wt");
    if (fp == NULL) {
        perror(__PRETTY_FUNCTION__);
        return false;
    }
    long l = 0;
    while (size > 0) {
        int len;
        if ((len = fprintf(fp, "hello! %ld\\n", l++)) < 0) {
            fclose(fp);
            perror(__PRETTY_FUNCTION__);
            return false;
        }
        size -= size > static_cast<size_t>(len) ? len : size;
    }
    fclose(fp);
    return true;
}
bool write_hello_by_stdcpp(const char* path, size_t size) noexcept {
    std::ofstream f(path, std::ios::out);
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    long l = 0;
    while (size > f.tellp()) {
        f << "hello! " << l++ << "\\n";
        if (! f) {
            std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
            return false;
        }
    }
    f.flush();
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    return true;
}
EOF
cat >test.cpp <<EOF
#include <chrono>
#include <cmath>
#include <cstdint>
#include <cassert>
#include "mitata.hpp"  // https://github.com/evanwashere/mitata/raw/refs/tags/v1.0.23/src/mitata.hpp
#define ANKERL_NANOBENCH_IMPLEMENT
#include "nanobench.h" // https://raw.githubusercontent.com/martinus/nanobench/v4.3.11/src/include/nanobench.h
#include "example.hpp"
int main() {
  //   std::ios_base::sync_with_stdio(false);
  {
    std::cout << "======================== mitata ========================" << std::endl;
    mitata::runner runner;
    runner.summary([&]() {
      runner.bench("standard C++", []() {assert(write_by_stdcpp("test.bin", 1024 * 1024 * 100));});
      runner.bench("standard C", []() {assert(write_by_stdc("test.bin", 1024 * 1024 * 100));});
      runner.bench("system call", []() {assert(write_by_systemcall("test.bin", 1024 * 1024 * 100));});
    });
    auto stats = runner.run();
  }
  {
    std::cout << "======================== nanobench ========================" << std::endl;
    ankerl::nanobench::Bench().run("standard C++", [&] {
      assert(write_by_stdcpp("test.bin", 1024 * 1024 * 100));
    });
    ankerl::nanobench::Bench().run("standard C", [&] {
      assert(write_by_stdc("test.bin", 1024 * 1024 * 100));
    });
    ankerl::nanobench::Bench().run("system call", [&] {
      assert(write_by_systemcall("test.bin", 1024 * 1024 * 100));
    });
  }
  {
    std::cout << "======================== mitata ========================" << std::endl;
    mitata::runner runner;
    runner.summary([&]() {
      runner.bench("standard C++", []() {assert(write_hello_by_stdcpp("test.bin", 1024 * 1024 * 100));});
      runner.bench("standard C", []() {assert(write_hello_by_stdc("test.bin", 1024 * 1024 * 100));});
    });
    auto stats = runner.run();
  }
  {
    std::cout << "======================== nanobench ========================" << std::endl;
    ankerl::nanobench::Bench().run("standard C++", [&] {
      assert(write_hello_by_stdcpp("test.bin", 1024 * 1024 * 100));
    });
    ankerl::nanobench::Bench().run("standard C", [&] {
      assert(write_hello_by_stdc("test.bin", 1024 * 1024 * 100));
    });
  }
  return 0;
}
EOF
wget 'https://github.com/evanwashere/mitata/raw/refs/tags/v1.0.23/src/mitata.hpp'
wget 'https://raw.githubusercontent.com/martinus/nanobench/v4.3.11/src/include/nanobench.h'
patch -p1 <<EOF
--- a/mitata.hpp	2025-02-05 14:54:25.926948220 +0900
@@ -217,11 +217,11 @@ namespace mitata {
       for (auto o = 0; o <= poffset; o++) bins[std::round((stats.samples[o] - min) / step)]++;
 
       return {
+        .avg = clamp(0, (u64)std::round((stats.avg - min) / step), size - 1),
+        .peak = *std::max_element(bins.begin(), bins.end()),
+        .outliers = stats.samples.size() - 1 - poffset,
         .min = min, .max = max,
         .step = step, .bins = bins, .steps = steps,
-        .outliers = stats.samples.size() - 1 - poffset,
-        .peak = *std::max_element(bins.begin(), bins.end()),
-        .avg = clamp(0, (u64)std::round((stats.avg - min) / step), size - 1),
       };
     }
EOF
g++ -shared -fPIC -g -Wall -pedantic -std=c++17 -O3 example.cpp -o libexample.so
g++ -c -g -Wall -std=c++17 -O3 test.cpp -o test.o
g++ test.o -L. -lexample -o test
LD_LIBRARY_PATH=$(pwd) ./test
```

----

# 2026年6月24日 - 追記

`write_hello_by_stdcpp()`は`tellp()`にかなりの時間を要する**極端に遅い**コードだった^[tellpで実際にseekするとは思っていなかったのですが、C側で動かされるケースを想定しているのか毎回seekが起きていました]ので、修正版のコードを`write_hello_by_stdcpp_without_tellp()`として用意しました。

別のPCでの測定になりますが、結果は以下のとおりになります。

```shell-session
benchmark                   avg (min … max) p75   p99    (min … top 1%)
------------------------------------------- -------------------------------
standard C                    25.48 ms/iter  26.09 ms ▅▂▅   ▂ ▂    █
                      (24.41 ms … 27.28 ms)  27.20 ms ███▁▄▁█▇█▇▄▇▇█▁▁▄▁▁▁▄
standard C++                  25.65 ms/iter  26.49 ms  █  ▄   ▄       █   ▄
                      (24.24 ms … 27.18 ms)  27.07 ms ▅█▅██▅▅▁█▁▅██▅█▅██▁▁█
system call                   25.36 ms/iter  25.88 ms   █▄   ▄  █ █ ▄     ▄
                      (24.15 ms … 26.63 ms)  26.62 ms ▅▅██▅▅██▁▅█▁█▁█▁▅▅▁██

summary
  system call
   1x faster than standard C
   1.01x faster than standard C++
======================== nanobench ========================

|               ns/op |                op/s |    err% |     total | benchmark
|--------------------:|--------------------:|--------:|----------:|:----------
|       25,485,073.00 |               39.24 |    3.6% |      0.28 | `standard C++`
|       24,629,874.00 |               40.60 |    1.5% |      0.27 | `standard C`
|       25,689,201.00 |               38.93 |    4.3% |      0.29 | `system call`
======================== mitata ========================
runtime: c++
compiler: gcc

benchmark                   avg (min … max) p75   p99    (min … top 1%)
------------------------------------------- -------------------------------
standard C                   195.88 ms/iter 197.08 ms         █    ██
                    (190.88 ms … 200.00 ms) 199.54 ms █▁▁▁▁█▁▁█▁▁████▁▁▁▁▁█
standard C++                 742.74 ms/iter 750.85 ms ▃    █          ▃
                    (730.65 ms … 760.23 ms) 756.12 ms █▆▁▁▁█▁▁▁▁▆▆▁▁▁▁█▁▁▁▆
standard C++ without tellp   202.69 ms/iter 202.85 ms        █            █
                    (199.72 ms … 207.55 ms) 205.57 ms █▁▁██▁███▁██▁▁▁▁▁▁▁▁█

summary
  standard C
   1.03x faster than standard C++ without tellp
   3.79x faster than standard C++
======================== nanobench ========================
|      738,583,639.00 |                1.35 |    0.6% |      8.11 | `standard C++`
|      202,258,382.00 |                4.94 |    1.4% |      2.23 | `standard C++ without tellp`
|      196,697,352.00 |                5.08 |    1.2% |      2.16 | `standard C`
```

修正版のコードではCとC++でほとんど差がない結果になっています。

付録に付けていた全体のコードに修正版を入れたものを以下に置いておきます。

```sh:test2.sh
set -eux
cat >example.hpp <<EOF
#pragma once
extern bool write_by_systemcall(const char* path, size_t size) noexcept;
extern bool write_by_stdc(const char* path, size_t size) noexcept;
extern bool write_by_stdcpp(const char* path, size_t size) noexcept;
extern bool write_hello_by_stdc(const char* path, size_t size) noexcept;
extern bool write_hello_by_stdcpp(const char* path, size_t size) noexcept;
EOF
cat >example.cpp <<EOF
#include <fcntl.h>
#include <unistd.h>
#include <cstdlib>
#include <vector>
#include <cstdio>
#include <iostream>
#include <ios>
#include <fstream>
#include "example.hpp"
const size_t BUFFER_SIZE = 1024 * 1024;
bool write_by_systemcall(const char* path, size_t size) noexcept {
    std::vector<char> buffer(BUFFER_SIZE);
    int fd = open(path, O_CREAT | O_WRONLY | O_TRUNC, 0666);
    if (fd == -1) {
        perror(__PRETTY_FUNCTION__);
        return false;
    }
    while (size > 0) {
        size_t len = buffer.size() > size ? size : buffer.size();
        if (write(fd, buffer.data(), len) != static_cast<ssize_t>(len)) {
            close(fd);
            perror(__PRETTY_FUNCTION__);
            return false;
        }
        size -= len;
    }
    close(fd);
    return true;
}
bool write_by_stdc(const char* path, size_t size) noexcept {
    std::vector<char> buffer(BUFFER_SIZE);
    FILE* fp = fopen(path, "wb");
    if (fp == NULL) {
        perror(__PRETTY_FUNCTION__);
        return false;
    }
    while (size > 0) {
        size_t len = buffer.size() > size ? size : buffer.size();
        if (fwrite(buffer.data(), sizeof(decltype(buffer)::value_type), len, fp) != static_cast<size_t>(len)) {
            fclose(fp);
            perror(__PRETTY_FUNCTION__);
            return false;
        }
        size -= len;
    }
    fclose(fp);
    return true;
}
bool write_by_stdcpp(const char* path, size_t size) noexcept {
    std::vector<char> buffer(BUFFER_SIZE);
    std::ofstream f(path, std::ios::out | std::ios::binary);
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    while (size > 0) {
        size_t len = buffer.size() > size ? size : buffer.size();
        f.write(buffer.data(), len);
        if (! f) {
            std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
            return false;
        }
        size -= len;
    }
    f.flush();
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    return true;
}
bool write_hello_by_stdc(const char* path, size_t size) noexcept {
    FILE* fp = fopen(path, "wt");
    if (fp == NULL) {
        perror(__PRETTY_FUNCTION__);
        return false;
    }
    long l = 0;
    while (size > 0) {
        int len;
        if ((len = fprintf(fp, "hello! %ld\\n", l++)) < 0) {
            fclose(fp);
            perror(__PRETTY_FUNCTION__);
            return false;
        }
        size -= size > static_cast<size_t>(len) ? len : size;
    }
    fclose(fp);
    return true;
}
bool write_hello_by_stdcpp(const char* path, size_t size) noexcept {
    std::ofstream f(path, std::ios::out);
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    long l = 0;
    while (size > f.tellp()) {
        f << "hello! " << l++ << "\\n";
        if (! f) {
            std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
            return false;
        }
    }
    f.flush();
    if (! f) {
        std::cerr << __PRETTY_FUNCTION__ << " failed" << std::endl;
        return false;
    }
    return true;
}
EOF
cat >test.cpp <<EOF
#include <chrono>
#include <cmath>
#include <cstdint>
#include <cassert>
#include "mitata.hpp"  // https://github.com/evanwashere/mitata/raw/refs/tags/v1.0.23/src/mitata.hpp
#define ANKERL_NANOBENCH_IMPLEMENT
#include "nanobench.h" // https://raw.githubusercontent.com/martinus/nanobench/v4.3.11/src/include/nanobench.h
#include "example.hpp"
int main() {
  //   std::ios_base::sync_with_stdio(false);
  {
    std::cout << "======================== mitata ========================" << std::endl;
    mitata::runner runner;
    runner.summary([&]() {
      runner.bench("standard C++", []() {assert(write_by_stdcpp("test.bin", 1024 * 1024 * 100));});
      runner.bench("standard C", []() {assert(write_by_stdc("test.bin", 1024 * 1024 * 100));});
      runner.bench("system call", []() {assert(write_by_systemcall("test.bin", 1024 * 1024 * 100));});
    });
    auto stats = runner.run();
  }
  {
    std::cout << "======================== nanobench ========================" << std::endl;
    ankerl::nanobench::Bench().run("standard C++", [&] {
      assert(write_by_stdcpp("test.bin", 1024 * 1024 * 100));
    });
    ankerl::nanobench::Bench().run("standard C", [&] {
      assert(write_by_stdc("test.bin", 1024 * 1024 * 100));
    });
    ankerl::nanobench::Bench().run("system call", [&] {
      assert(write_by_systemcall("test.bin", 1024 * 1024 * 100));
    });
  }
  {
    std::cout << "======================== mitata ========================" << std::endl;
    mitata::runner runner;
    runner.summary([&]() {
      runner.bench("standard C++", []() {assert(write_hello_by_stdcpp("test.bin", 1024 * 1024 * 100));});
      runner.bench("standard C", []() {assert(write_hello_by_stdc("test.bin", 1024 * 1024 * 100));});
    });
    auto stats = runner.run();
  }
  {
    std::cout << "======================== nanobench ========================" << std::endl;
    ankerl::nanobench::Bench().run("standard C++", [&] {
      assert(write_hello_by_stdcpp("test.bin", 1024 * 1024 * 100));
    });
    ankerl::nanobench::Bench().run("standard C", [&] {
      assert(write_hello_by_stdc("test.bin", 1024 * 1024 * 100));
    });
  }
  return 0;
}
EOF
wget 'https://github.com/evanwashere/mitata/raw/refs/tags/v1.0.23/src/mitata.hpp'
wget 'https://raw.githubusercontent.com/martinus/nanobench/v4.3.11/src/include/nanobench.h'
patch -p1 <<EOF
--- a/mitata.hpp	2025-02-05 14:54:25.926948220 +0900
@@ -217,11 +217,11 @@ namespace mitata {
       for (auto o = 0; o <= poffset; o++) bins[std::round((stats.samples[o] - min) / step)]++;
 
       return {
+        .avg = clamp(0, (u64)std::round((stats.avg - min) / step), size - 1),
+        .peak = *std::max_element(bins.begin(), bins.end()),
+        .outliers = stats.samples.size() - 1 - poffset,
         .min = min, .max = max,
         .step = step, .bins = bins, .steps = steps,
-        .outliers = stats.samples.size() - 1 - poffset,
-        .peak = *std::max_element(bins.begin(), bins.end()),
-        .avg = clamp(0, (u64)std::round((stats.avg - min) / step), size - 1),
       };
     }
EOF
g++ -shared -fPIC -g -Wall -pedantic -std=c++17 -O3 example.cpp -o libexample.so
g++ -c -g -Wall -std=c++17 -O3 test.cpp -o test.o
g++ test.o -L. -lexample -o test
LD_LIBRARY_PATH=$(pwd) ./test
```

