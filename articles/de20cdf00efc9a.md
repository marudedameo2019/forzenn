---
title: "C++ Core Guidelinesでチェックする"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cpp","linux","docker"]
published: false
---

# 序

いきなり余談ですが、先日、Bjarne Stroustrup先生が警鐘を鳴らしていたとされる記事を見かけました。

https://www.theregister.com/2025/03/02/c_creator_calls_for_action/

最近この人何してるんだろ？と検索してみたところ、

https://www.youtube.com/watch?v=G7oY8QVL3Fs

という動画が出てきて、個人的にはあまり感想もないインタビューだったのですが、その中で11:30くらいでC++のガイドラインがあるよという話が出てきました。これでどんなチェックが出来るのかを見てみることが、この記事の目的です。ガイドライン自体は以下になります。

https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines

このガイドラインにはチェックツールがあり、

https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#tools-clang-tidy

によると、Linuxではclang-tidyのようです。

https://clang.llvm.org/extra/clang-tidy/index.html

※Linuxを選んでいるのは、dockerを使うことで多くのOSで動かせるからです

# 1. 前提

対象読者は以下の方々です。

- C++をご存知の方
- Linuxを使える方
- dockerを使える方

この前提に従い、これらに関する初歩的な説明は割愛させていただきます。

# 2. 環境構築

dockerを使用し、clang-tidyが動く環境を用意します。と言っても、ubuntuにclang-tidyを入れるだけですが…

```docker:Dockerfile
FROM ubuntu:24.04
RUN apt-get update && \
    apt-get upgrade --no-install-recommends -y && \
    apt-get install --no-install-recommends -y clang-tidy
```

```sh:build.sh
docker build -t clang_tidy .
```

```shell_session
$ sh build.sh
...
$
```

これでclang-tidyイメージの完成です。あとはこのイメージを使ったチェックが出来ればOKです。

```sh:check.sh
UID=$(id -u)
GID=$(id -g)
PWD=$(pwd)
USERNAME=$(id -nu)
docker run -it --rm \
    -u $UID:$GID \
    -v $PWD:/home/$USERNAME \
    -w /home/$USERNAME \
    -e HOME=/home/$USERNAME \
    clang_tidy \
    clang-tidy -checks='cppcoreguidelines-*' $* -- -std=c++17
```

`sh check.sh ファイル名`でチェックできます。

# 3. ガイドラインによるチェック

## 3-1. チェック対象コード

何でも良かったのですが、昔適当に書いた記事に付いてるコードを試してみました。

https://zenn.dev/dameyodamedame/articles/59cab0f2c975b0

記事内に2つコードがあるので、2つともチェックしてみました。

## 3-2. チェック結果

### 3-2-1. input_timeout.cpp

最初のコードです。

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/input_timeout.cpp

```shell_session
$ sh check.sh input_timeout.cpp
...
```

以下ではそのチェック結果を1つずつ見ていきます。

#### 3-2-1-1. macro-usage

https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/macro-usage.html

```plain
/home/user/input_timeout.cpp:13:9: warning: function-like macro 'THROW_EXCEPTION' used; consider a 'constexpr' template function [cppcoreguidelines-macro-usage]
   13 | #define THROW_EXCEPTION(s) throw runtime_error(string(s ": ") + strerror(errno))
      |         ^
```

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/input_timeout.cpp#L13

これはすこぶる付きで大変な感じですが、

```sh:cppver.sh
set -ex
cat >cppver.cpp <<EOF
#include <iostream>
int main() { std::cout << __cplusplus << std::endl; }
EOF
make cppver
./cppver
```

```shell_session
$ sh cppver.sh
+ cat
+ make cppver
g++     cppver.cpp   -o cppver
+ ./cppver
201703
$
```

という感じで、C++17を想定してるので、std::arrayとstd::string_viewを使って

```cpp
#include <array>
#include <string_view>

template <const std::string_view& S1, const std::string_view& S2>
struct concat {
    static constexpr auto init() noexcept {
        constexpr std::size_t len = S1.size() + S2.size() + 1;
        std::array<char, len> ar{};
        auto append = [i=0, &ar](const auto& s) mutable { for (auto e : s) ar.at(i++) = e; };
        append(S1);
        append(S2);
        ar[len-1] = 0;
        return ar;
    }
    static constexpr auto arr = init();
    static constexpr std::string_view value{arr.data(), arr.size()-1};
};

template <const std::string_view& S>
void throw_exception() {
    static constexpr std::string_view colon_space = ": ";
    throw std::runtime_error(std::string(concat<S, colon_space>::value) + std::strerror(errno));
}
...
    try {
        static constexpr std::string_view xxx = "xxx";
        throw_exception<xxx>();
    }
    catch(std::runtime_error& e) {
        std::cerr << e.what() << std::endl;
    }
```

こうすれば同じ動作になるはずです。throw_exceptionテンプレート関数がマクロの代わりなのですが、実装がかなり面倒な上に使い方も面倒になっており、比較的改悪だと思います(コツは静的に配置することで定数としテンプレート引数で渡すこと)。C++20ならもう少しマシになり、C++17未満だともっと大変になると思います。

__参考にしたコード__

https://stackoverflow.com/questions/38955940/how-to-concatenate-static-strings-at-compile-time/62823211#62823211

#### 3-2-1-2. special-member-functions

https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/special-member-functions.html
```plain
/home/user/input_timeout.cpp:15:8: warning: class 'auto_close_fd' defines a non-default destructor but does not define a copy constructor, a copy assignment operator, a move constructor or a move assignment operator [cppcoreguidelines-special-member-functions]
   15 | struct auto_close_fd {
      |        ^
```

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/input_timeout.cpp#L15-L20

こんな感じでデストラクタ作ったなら他もちゃんと定義してねと言われてます。仰るとおりなので、とりあえずdeleteで宣言しておきます。

```cpp
struct auto_close_fd {
    int fd;
    operator int() {return fd;}
    auto_close_fd(int fd): fd(fd) {}
    auto_close_fd(const auto_close_fd&) = delete;
    auto_close_fd(auto_close_fd&&) = delete;
    auto_close_fd& operator=(const auto_close_fd&) = delete;
    auto_close_fd& operator=(auto_close_fd&&) = delete;
    ~auto_close_fd() {if (fd != -1) close(fd);}
};
```
#### 3-2-1-3. pro-type-member-init

https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/pro-type-member-init.html

```plain
/home/user/input_timeout.cpp:27:9: warning: uninitialized record type: 'ev' [cppcoreguidelines-pro-type-member-init]
   27 |         epoll_event ev;
      |         ^             
```

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/input_timeout.cpp#L27

初期化されてないとのことなので、しておきます。

```cpp
        epoll_event ev{};
```

#### 3-2-1-4. avoid-c-arrays

https://clang.llvm.org/extra/clang-tidy/checks/modernize/avoid-c-arrays.html

```plain
/home/user/input_timeout.cpp:32:9: warning: do not declare C-style arrays, use std::array<> instead [cppcoreguidelines-avoid-c-arrays]
   32 |         epoll_event evs[1];
      |         ^
```

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/input_timeout.cpp#L32

Cスタイルの配列宣言は(暗黙的にポインタに変換され、境界情報が欠損するので)やめとけ、とのことなので、std::arrayに変えときます。

```cpp
        std::array<epoll_event,1> evs;
```

__別の箇所にある同じ警告__

```plain
/home/user/input_timeout.cpp:40:13: warning: do not declare C-style arrays, use std::array<> instead [cppcoreguidelines-avoid-c-arrays]
   40 |             char buffer[10];
      |             ^
```

#### 3-2-1-5. pro-bounds-array-to-pointer-decay

https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/pro-bounds-array-to-pointer-decay.html

```plain
/home/user/input_timeout.cpp:33:37: warning: do not implicitly decay an array into a pointer; consider using gsl::array_view or an explicit cast instead [cppcoreguidelines-pro-bounds-array-to-pointer-decay]
   33 |         auto len = epoll_wait(epfd, evs, sizeof(evs)/sizeof(evs[0]), 10000);
      |                                     ^
```

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/input_timeout.cpp#L33

一つ上のやつと同じ理由ですね。上の対策でstd::arrayにすれば自然と消えます。

__別の箇所にある同じ警告__

```plain
/home/user/input_timeout.cpp:41:51: warning: do not implicitly decay an array into a pointer; consider using gsl::array_view or an explicit cast instead [cppcoreguidelines-pro-bounds-array-to-pointer-decay]
   41 |             auto read_size = read(evs[0].data.fd, buffer, sizeof(buffer));
      |                                                   ^
/home/user/input_timeout.cpp:43:42: warning: do not implicitly decay an array into a pointer; consider using gsl::array_view or an explicit cast instead [cppcoreguidelines-pro-bounds-array-to-pointer-decay]
   43 |             cout << "read: \"" << string(buffer, read_size) << "\"" << endl;
      |         
```

#### 3-2-1-6. avoid-magic-numbers

https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/avoid-magic-numbers.html

```plain
/home/user/input_timeout.cpp:40:25: warning: 10 is a magic number; consider replacing it with a named constant [cppcoreguidelines-avoid-magic-numbers]
   40 |             char buffer[10];
      |                         ^
```

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/input_timeout.cpp#L40

昔からあるやつですね。個人的には1つしかない短いコードの場合は却って追いにくくなるので、マジックナンバーにしちゃうやつです。直す場合はconstexpr推奨のようです。

```cpp
            constexpr std::size_t BUFFER_SIZE = 10;
            char buffer[BUFFER_SIZE];
```

普通に直すとこうなんですが、これ結局Cスタイルの配列宣言だって怒られるので、こうしておきます。

```cpp
            constexpr std::size_t BUFFER_SIZE = 10;
            std::array<char,BUFFER_SIZE> buffer{};
```

### 3-2-2. event_loop_example.cpp

2番目のコードです。

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/event_loop_example.cpp

#### 3-2-2-1. noexcept-move-operations

https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/noexcept-move-operations.html
https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-move-noexcept

```plain
/home/user/event_loop_example.cpp:17:5: warning: move constructors should be marked noexcept [cppcoreguidelines-noexcept-move-operations]
   17 |     file_descriptor(file_descriptor &&org): fd_(org.fd_) {org.fd_ = INVALID;}
      |     ^                                     
      |                                            noexcept 
```

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/event_loop_example.cpp#L17

ムーブ操作はnoexceptにしないと効率が悪い(コピーを選ばれてしまう(未確認))という教えなので、従うべきでしょう。パフォーマンス第一なら本来全てに記述すべき(true/false記述)ですしね。

```cpp
    file_descriptor(file_descriptor &&org) noexcept: fd_(org.fd_) {org.fd_ = INVALID;}
```

__別の箇所にある同じ警告__

```plain
/home/user/event_loop_example.cpp:19:22: warning: move assignment operators should be marked noexcept [cppcoreguidelines-noexcept-move-operations]
   19 |     file_descriptor &operator=(file_descriptor &&org) {
      |                      ^
      |                                                        noexcept 
```

#### 3-2-2-2. special-member-functions

[3-2-1-2. special-member-functions](#3-2-1-2.-special-member-functions) と同様

__別の箇所にある同じ警告__

```plain
/home/user/event_loop_example.cpp:35:7: warning: class 'epoll' defines a copy constructor, a copy assignment operator, a move constructor and a move assignment operator but does not define a destructor [cppcoreguidelines-special-member-functions]
   35 | class epoll {
      |       ^
```
#### 3-2-2-3. pro-type-member-init

[3-2-1-3. pro-type-member-init](#3-2-1-3-pro-type-member-init)と同様

__別の箇所にある同じ警告__

```plain
/home/user/event_loop_example.cpp:44:9: warning: uninitialized record type: 'ev' [cppcoreguidelines-pro-type-member-init]
   44 |         epoll_event ev;
      |         ^             
      |                       {}
/home/user/event_loop_example.cpp:74:9: warning: uninitialized record type: 'buff' [cppcoreguidelines-pro-type-member-init]
   74 |         buff8bytes buff;
      |         ^              
      |                        {}
/home/user/event_loop_example.cpp:121:25: warning: uninitialized record type: 'buff' [cppcoreguidelines-pro-type-member-init]
  121 |                         buff8bytes buff;
      |                         ^              
      |                                        {}
```

#### 3-2-2-4. narrowing-conversions

https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/narrowing-conversions.html

```plain
/home/user/event_loop_example.cpp:57:64: warning: narrowing conversion from 'size_type' (aka 'unsigned long') to signed type 'int' is implementation-defined [cppcoreguidelines-narrowing-conversions]
   57 |         auto len = epoll_wait(this->fd_, this->events_.data(), this->events_.size(), timeout);
      |                                                                ^
```

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/event_loop_example.cpp#L57

これもよくあるキャストにより表現範囲が小さくなってしまう問題ですね。意図した縮退であることを示すため、明示的にstatic_castすれば大丈夫です。サポートライブラリを使うと、例外を投げるようにも実装できます。ここは実装ロジックから定数にもできます。

```cpp
        auto len = epoll_wait(this->fd_, this->events_.data(), static_cast<int>(this->events_.size()), timeout);
```

__別の箇所にある同じ警告__

```plain
/home/user/event_loop_example.cpp:117:39: warning: narrowing conversion from 'rep' (aka 'long') to signed type 'int' is implementation-defined [cppcoreguidelines-narrowing-conversions]
  117 |             auto len = this->ep_.wait(ms);
      |                                       ^
/home/user/event_loop_example.cpp:146:33: warning: narrowing conversion from 'size_type' (aka 'unsigned long') to signed type 'streamsize' (aka 'long') is implementation-defined [cppcoreguidelines-narrowing-conversions]
  146 |         cout.write(buff.data(), buff.size());
      |                                 ^
```
#### 3-2-2-5. avoid-c-arrays

[3-2-1-4. avoid-c-arrays](#3-2-1-4-avoid-c-arrays)と同様

__別の箇所にある同じ警告__

```plain
/home/user/event_loop_example.cpp:70:9: warning: do not declare C-style arrays, use std::array<> instead [cppcoreguidelines-avoid-c-arrays]
   70 |         char bytes[8];
      |         ^
/home/user/event_loop_example.cpp:162:22: warning: do not declare C-style arrays, use std::array<> instead [cppcoreguidelines-avoid-c-arrays]
  162 |         static const char pre_line[] = {"\x1b[s\x1b[1A\r"};
      |                      ^
/home/user/event_loop_example.cpp:163:22: warning: do not declare C-style arrays, use std::array<> instead [cppcoreguidelines-avoid-c-arrays]
  163 |         static const char post_line[] = {"\x1b[u"};
      |                      ^
/home/user/event_loop_example.cpp:164:22: warning: do not declare C-style arrays, use std::array<> instead [cppcoreguidelines-avoid-c-arrays]
  164 |         static const char* rest_chars[] = {" ", "\u258F", "\u258E", "\u258D", "\u258C", "\u258B", "\u258A", "\u2589"};
      |                      ^
```

#### 3-2-2-6. avoid-magic-numbers

[3-2-1-6. avoid-magic-numbers](#3-2-1-6-avoid-magic-numbers)と同様

__別の箇所にある同じ警告__

```plain
/home/user/event_loop_example.cpp:70:20: warning: 8 is a magic number; consider replacing it with a named constant [cppcoreguidelines-avoid-magic-numbers]
   70 |         char bytes[8];
      |                    ^
/home/user/event_loop_example.cpp:168:42: warning: 8 is a magic number; consider replacing it with a named constant [cppcoreguidelines-avoid-magic-numbers]
  168 |         auto char8size = ms * width100 * 8 / WAIT_TIME_MS;
      |                                          ^
```

#### 3-2-2-7. pro-type-union-access

https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/pro-type-union-access.html
https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Ru-naked

```plain
/home/user/event_loop_example.cpp:75:14: warning: do not access members of unions; use (boost::)variant instead [cppcoreguidelines-pro-type-union-access]
   75 |         buff.value = 1;
      |              ^
```

https://github.com/marudedameo2019/trying_out_epoll/blob/bed0e3f5db9eec83303ae27722db5acf436a2605/event_loop_example.cpp#L75

これは型安全のためにunionは使わないでくれ、というものでstd::variantを代替で提案しています。ただ、このケースでは各メンバが全く同じアドレスであることを前提としたunion利用で、もっと言えばアラインメントやバイトオーダーも普通の処理系はこういうの動くようになってるよね、的な楽観的予想に基づいた原始的なコードなので、std::variantでは代替できません(生成する型と参照する型が異なるのでassertionに引っかかる)。なので、今回この型に関連する警告は無視([NOLINT](https://clang.llvm.org/extra/clang-tidy/index.html#suppressing-undesired-diagnostics))しています。

```cpp
        buff.value = 1; // NOLINT(cppcoreguidelines-pro-type-union-access)
```

__別の箇所にある同じ警告__

```plain
/home/user/event_loop_example.cpp:76:32: warning: do not access members of unions; use (boost::)variant instead [cppcoreguidelines-pro-type-union-access]
   76 |         write(this->fd_, &buff.bytes[0], sizeof(buff.bytes));
      |                                ^
/home/user/event_loop_example.cpp:76:54: warning: do not access members of unions; use (boost::)variant instead [cppcoreguidelines-pro-type-union-access]
   76 |         write(this->fd_, &buff.bytes[0], sizeof(buff.bytes));
      |                                                      ^
/home/user/event_loop_example.cpp:122:47: warning: do not access members of unions; use (boost::)variant instead [cppcoreguidelines-pro-type-union-access]
  122 |                         read(e.data.fd, &buff.bytes[0], sizeof(buff.bytes));
      |                                               ^
/home/user/event_loop_example.cpp:122:69: warning: do not access members of unions; use (boost::)variant instead [cppcoreguidelines-pro-type-union-access]
  122 |                         read(e.data.fd, &buff.bytes[0], sizeof(buff.bytes));
      |                                                                     ^
```
#### 3-2-2-8. pro-bounds-array-to-pointer-decay

[3-2-1-5. pro-bounds-array-to-pointer-decay](#3-2-1-5-pro-bounds-array-to-pointer-decay)と同様

__別の箇所にある同じ警告__

```plain
/home/user/event_loop_example.cpp:171:17: warning: do not implicitly decay an array into a pointer; consider using gsl::array_view or an explicit cast instead [cppcoreguidelines-pro-bounds-array-to-pointer-decay]
  171 |         cout << pre_line;
      |                 ^
/home/user/event_loop_example.cpp:176:17: warning: do not implicitly decay an array into a pointer; consider using gsl::array_view or an explicit cast instead [cppcoreguidelines-pro-bounds-array-to-pointer-decay]
  176 |         cout << post_line;
      |                 ^
```

### 3-3. 修正結果

以下が修正差分になります。

https://github.com/marudedameo2019/trying_out_epoll/commit/8f657fe3e96a3031d3f03786b937129d4309e7b4

zennで当然綺麗に整形されると思ったけど、出来なかったので、リンクのみです。埋め込みないとダメなのかも。

# 4. まとめ

既存のコードをclang-tidyを使ってC++ Core Guidelinesでチェックしてみたところ、かなりの箇所が引っかかった。
どうしようもないところもあったが、修正を行い、コードの質を上げることが出来た。

|指標|値|補足|
|--|--|--|
|LoC|233|[4-1-1. LoC算出](#4-1-1-loc算出)参照|
|指摘の数|32|[4-1-2. 指摘数抽出](#4-1-2-指摘数抽出)参照|

|チェック|回数|
|--|--|
|avoid-c-arrays|6|
|pro-type-union-access|5|
|pro-bounds-array-to-pointer-decay|5|
|pro-type-member-init|4|
|avoid-magic-numbers|4|
|narrowing-conversions|3|
|special-member-functions|2|
|noexcept-move-operations|2|
|macro-usage|1|

※[4-1-3. 各チェックの頻度サマリ](#4-1-3-各チェックの頻度サマリ)参照

## 4-1. 補足

### 4-1-1. LoC算出

```sh
wc -l input_timeout.cpp event_loop_example.cpp
```

### 4-1-2. 指摘数抽出

```sh
for f in input_timeout.cpp event_loop_example.cpp;do \
   sh check.sh $f;\
done | \
sed -e 's/\x1b\[[0-9;]*m//g' | \
grep -P '^/.*\.cpp:\d+:\d+: warning: ' | \
wc -l
```

### 4-1-3. 各チェックの頻度サマリ

```sh
for f in input_timeout.cpp event_loop_example.cpp;do \
   sh check.sh $f; \
done | \
sed -e 's/\x1b\[[0-9;]*m//g;s/\r$//' | \
grep -P '^/.*\.cpp:\d+:\d+: warning: ' | \
sed 's/.*\[cppcoreguidelines-\(.*\)\]/\1/' | \
sort | \
uniq -c | \
sort -k 1 -rn | \
awk '{print "|" $2 "|" $1 "|";}'
```