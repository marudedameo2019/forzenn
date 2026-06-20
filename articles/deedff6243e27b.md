---
title: "MS-DOS 4.0をビルドして動かす"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MSDOS"]
published: true
---
# 序

最古のDOS、PC-DOS 1.00のソースコード資料などが最近公開されたのを知り^[[Continuing the story of early DOS development | Microsoft Open Source Blog](https://opensource.microsoft.com/blog/2026/04/28/continuing-the-story-of-early-dos-development/)]、そのとき初めてMS-DOSのソースコードがVer4.0まで既に公開されている^[[microsoft/MS-DOS: The original sources of MS-DOS 1.25, 2.0, and 4.0 for reference purposes](https://github.com/microsoft/MS-DOS)]ことを知りました。

古いとはいえ、エミュレータではない本物のDOSがソースコード付きで出てくるのであれば、気になって調べてみた次第です。見るとどうやらMS-CもMASMもCLIだけなら付いてくるようで、ソースはあっても普通にはビルドできない、なんてこともないようです。大盤振る舞いな点に惹かれてとりあえずQEMUで動かすところまでは漕ぎつけました。次章以降ではその手順をご説明いたします。

# 1. 必要なもの

エミュレータを使う話なので、実際環境は問いません。ただし、今回の説明ではWindowsのMSYS2環境(UCRT64)を前提に進めていきます。

- Windows
- MSYS2環境

MSYS2環境で必要なパッケージは、以下のとおりです。

- mingw-w64-ucrt-x86_64-qemu
- dosfstools
- mingw-w64-ucrt-x86_64-mtools
- mingw-w64-ucrt-x86_64-dosbox-staging

辺りでした(dosfstoolsはdosfsckで使っただけなので、必要ないかも)。

# 2. ソースを取ってくる

```sh
$ git clone https://github.com/microsoft/MS-DOS.git
...
$ 
```

# 3. パッチを当てる

https://haoict.github.io/operating-systems/windows/build-msdos-4.00-from-source/

以降では上記を参考に、パッチを当てていきます。パッチの内容は以下のとおりです。

- 改行コードを変更する
- US-ASCIIでコンパイル不可能な文字を修正する
- Cコードのinclude/lib関連の環境変数をこの構成に適応させる

パッチは〇nixライク系とPowershellの2種類が用意されているのですが、今回は〇nixライク系をベースにします。今回の環境はWindowsではあるものの、MSYS2を使うのでgit(MSYS/git)で取得すると改行はLFになるし^[commit時の方法に依存しますが]、ここで使用されているツールが使えるからです。

ただし件の記事にあるパッチスクリプトをわずかに修正して使います。修正したものが以下になります。

```sh:patch.sh
#!/usr/bin/env bash
set -e

echo "== Fix SETENV.BAT paths =="
sed \
  -i \
  -e 's|tools\\lib|tools\\bld\\lib|g' \
  -e 's|tools\\inc|tools\\bld\\inc|g' \
  SETENV.BAT

echo "== Replace bad characters with # =="
# These bytes: EF BF BD, C4 BF, C4 B4
for f in \
  MAPPER/GETMSG.ASM \
  SELECT/SELECT2.ASM \
  SELECT/USA.INF
do
  echo "Fixing $f"
  # Use perl for byte-level replacement (more reliable than sed here)
  perl -pi -e 's/\xEF\xBF\xBD|\xC4\xBF|\xC4\xB4/#/g' "$f"
done

echo "== Fix line endings to CRLF =="

fix_crlf \
() {
  local file="$1"
  echo "Fixing $file"
  # Convert LF -> CRLF safely (avoid doubling CR)
  perl -Mopen=IO,:raw -pi -e 's/(?<!\r)\n/\r\n/g' "$file"
}

export -f fix_crlf

# Process file types
find \
 . -type f \( \
    -iname "*.bat" -o \
    -iname "*.asm" -o \
    -iname "*.skl" -o \
    -iname "ZERO.DAT" -o \
    -iname "LOCSCR" \
\) -print0 | while IFS= read -r -d '' f; do
    fix_crlf "$f"
done

echo "== Done =="
```

変更点は以下の2つです。

- 改行抑制の行末`\`追加
- perlでテキストフィルタする際、cygwin系の自動改行変換を抑制するオプションを指定^[これを入れないと二重に改行変換されてCRCRLFが生成される]

改行変換は`unix2dos`みたいなものを使う方が分かりやすいのですが、オリジナルをなるべく変更しないように、またデフォルトのインストール状態で処理できるように、今回はperlを使って実施しています。

使い方は以下のとおりです。

```sh
$ pushd MS-DOS/v4.0/src
$ bash ../../../patch.sh
$ popd
```

# 4. ビルドをする

## 4-1. ディレクトリ移動

```sh
$ pushd MS-DOS/v4.0
```

## 4-2. dosbox-stagingを起動

MS-DOSをビルドするにはdosの環境が必要です。ビルドする前には当然ないので、この環境はエミュレータに任せます。選択肢はいくつかあるのですが、今回使用するのはMSYS2環境でパッケージが用意されている`dosbox-staging`です。

```sh
$ dosbox
...
```

こうすると、普通に起動します。そのままでも使えるのですが、すぐ`exit`してください。理由は普通に起動させると、ゲームなどでタイミングがおかしくならないように、デフォルトでは大きくウェイトが入っているからです。

今回はそれを外し最大パフォーマンスで動かしてみます。

```sh
$ dosbox --set core=dynamic --set cycles=max
...
```

最初のが64bitコードにJITコンパイルするモードで、次のがno waitなモード的な意味だそうです。MS-CやMASMを使ってビルドするならこちらの方が数十倍速いです。

## 4-3. ビルド

起動したdosbox上での作業になります。英語キーボードなので、いつものように頭の中で変換して打ちます(設定はあるが、私の環境では効かなかった)。

```cmd
Z:\>mount d .
Z:\>d:
D:\>cd src
D:\SRC>setenv
...
D:\SRC>nmake
...
D:\SRC>mkdir D:\BIN
D:\SRC>CPY.BAT D:\BIN
```

ビルド完了です。最高パフォーマンスでない場合(普通に起動した場合)、何十分もかかるので注意してください。出来上がったバイナリは`D:\BIN`にあります。

## 4-4. dosbox上でビルドしたバイナリを確認

この作業は確認なので必須ではありません。

```cmd
D:\SRC>cd \BIN
D:\BIN>ver set 4
D:\BIN>COMMAND.COM
...
D:>ver
MS-DOS Version 4.00
D:>exit
D:\BIN>exit
```

command.comからverコマンドしたときに`MS-DOS Version 4.00`みたいな表示が出ていればOKです。これはdosbox上の`COMMAND.COM`とは別のバイナリである証拠です。
ただこの状態は、dosboxが用意したOSシステムコールを使って、MS-DOSのCOMMAND.COMが動作している状態なので、正確にはMS-DOS 4.0上で動作していません。

## 4-5. ディレクトリを元に戻す

```sh
$ popd
```

# 5. QEMU上で動作させる

ビルドしたバイナリを使って、QEMU上でMS-DOS 4.0を動かします。

## 5-1. 起動用フロッピーディスクイメージの作成

まずはフロッピーディスクから起動出来るようにするのが当時の流儀です。方法は大きく2つ。

- DOS類似実行環境に仮想フロッピーイメージを入れて`format a: /s`でフォーマット
- 仮想フロッピーイメージをMSYS2上で自作

前者はDOS類似実行環境はdosboxがあるので、行けるかなと思ってたのですが、空の2HDディスクイメージをmountするところからもうダメっぽくて、色々試したのですが、断念しました^[freedosを使おうかとも思ったのですが、dosboxなら速いしMSYS2で入るし2つ使うのも嫌だったので、断念しました]。

今回は後者です。`SYS`コマンドのソースを解析して、セクタ0だけ固定値で用意し、他はMSYS2の`dd`、`dosfstools`、`mtools`を使ってディスクイメージを作ってみます。

まずはセクタ0部分はブート機能なので後回しにし、**起動しないイメージ**を作ります。

```sh
$ # 空の2HDディスクイメージ作成
$ dd if=/dev/zero of=floppy.img bs=1024 count=1440
$ # ディスクイメージをFAT12でフォーマット
$ mkfs.vfat -F 12 floppy.img
$ # ディスクイメージ内にIO.SYSをコピー
$ mcopy -i floppy.img MS-DOS/v4.0/BIN/IO.SYS ::
$ # ディスクイメージ内にMSDOS.SYSをコピー
$ mcopy -i floppy.img MS-DOS/v4.0/BIN/MSDOS.SYS ::
$ # ディスクイメージ内にCOMMAND.COMをコピー
$ mcopy -i floppy.img MS-DOS/v4.0/BIN/COMMAND.COM ::
```

ここまでで**起動しないイメージ**の完成です^[mkfs.vfatでボリュームラベルを指定すると起動しなくなるので注意]。

セクタ0イメージは以下のpythonコードで作ります。

```python:create_boot_bin.py
import os
import re

input_filename = "MS-DOS/v4.0/src/INC/BOOT.INC"
output_filename = "boot.bin"

if not os.path.exists(input_filename):
    print(f"Error: {input_filename} not found.")
    exit(1)

boot_bytes = bytearray()

try:
    with open(input_filename, "r", encoding="utf-8") as f:
        for line_num, line in enumerate(f, 1):
            line = line.split(";")[0].strip()
            if not line:
                continue

            matches = list(re.finditer(r'(?i)^\bdb\b|0?([0-9A-Fa-f]{2})H?\b', line))

            if not matches or matches[0].group(0).lower() != 'db':
                print(f"Syntax Error (Line {line_num}): Line must start with 'db'.")
                exit(1)

            line_data_len = 0
            for m in matches:
                h = m.group(1)
                if h is not None:
                    boot_bytes.append(int(h, 16))
                    line_data_len += 1

            if line_data_len == 0:
                print(f"Error (Line {line_num}): No valid 16-bit hex data found after 'db'.")
                exit(1)

except UnicodeDecodeError as e:
    print(f"File Error: Failed to decode {input_filename}. The file might not be encoded in UTF-8.\nDetails: {e}")
    exit(1)
except IOError as e:
    print(f"File Error: Cannot read {input_filename}. ({e})")
    exit(1)

print(f"Extracted: {len(boot_bytes)} bytes.")

def change_for_floppy(boot_bytes):
    # Bytes per Secotr: 0x200(512)
    boot_bytes[0x0b] = 0x00
    boot_bytes[0x0c] = 0x02
    # Sectors per Cluster: 0x01(1)
    boot_bytes[0x0d] = 0x01
    # Reserved Sectors: 0x0001(1)
    boot_bytes[0x0e] = 0x01
    boot_bytes[0x0f] = 0x00
    # Number of FATs: 0x02(2)
    boot_bytes[0x10] = 0x02
    # Root Entries: 0x00E0(224)
    boot_bytes[0x11] = 0xe0
    boot_bytes[0x12] = 0x00
    # Total Sectors 16bit: 0x0B40(2,880)
    boot_bytes[0x13] = 0x40
    boot_bytes[0x14] = 0x0b
    # Media Descriptor: 0xF0(240)
    boot_bytes[0x15] = 0xf0
    # Sectors per FAT: 0x0009(9)
    boot_bytes[0x16] = 0x09
    boot_bytes[0x17] = 0x00
    # Sectors per Track: 0x0012(18)
    boot_bytes[0x18] = 0x12
    boot_bytes[0x19] = 0x00
    # Number of Heads: 0x0002(2)
    boot_bytes[0x1a] = 0x02
    boot_bytes[0x1b] = 0x00
    # Hidden Sectors: 0x00000000(0)
    boot_bytes[0x1c] = 0x00
    boot_bytes[0x1d] = 0x00
    boot_bytes[0x1e] = 0x00
    boot_bytes[0x1f] = 0x00
    # Total Sectors 32bit: 0x00000000(0)
    boot_bytes[0x20] = 0x00
    boot_bytes[0x21] = 0x00
    boot_bytes[0x22] = 0x00
    boot_bytes[0x23] = 0x00

    # drive number
    boot_bytes[0x024] = 0

change_for_floppy(boot_bytes)
print("Changed for floppy.")

if len(boot_bytes) == 512:
    try:
        with open(output_filename, "wb") as f_out:
            f_out.write(boot_bytes)
        print(f"Success: '{output_filename}' (512 bytes) created.")
    except IOError as e:
        print(f"File Error: Cannot write to {output_filename}. ({e})")
        exit(1)
else:
    print(f"Error: Total data size is {len(boot_bytes)} bytes. It must be exactly 512 bytes.")
    exit(1)
```

input_filenameに指定された単純データファイルを読んで、中のデータをバイナリにした後、多少フロッピー用に書き換えて書き出すだけのコードです。「フロッピー用に書き換え」の部分以外は匿名geminiさんに書いてもらいました。あまり読んでません。

以下のように実行すると(要python3)、boot.binというファイルが作成されます。これがセクタ0の内容になります。

```sh
$ python3 create_boot_bin.py
```

これをセクタ0に書き込みます。

```sh
$ dd if=boot.bin of=floppy.img bs=512 count=1 conv=notrunc
```

ではまず、qemuで起動確認します。

```sh
$ qemu-system-i386 -drive file=floppy.img,format=raw,if=floppy
```

日付を聞かれたら成功です。Enter2回で動かせますが、実質command.comしかない状態なので、内部コマンドしか使えません。一旦qemuを閉じてください。

dosフォルダを作って残りのファイルをコピーします。

```sh
$ mmd -i floppy.img ::DOS
$ shopt -s extglob
$ mcopy -i floppy.img MS-DOS/v4.0/BIN/!(IO.SYS|MSDOS.SYS|COMMAND.COM) ::DOS
$ shopt -u extglob
```

再びqemuで起動確認します。

```sh
$ qemu-system-i386 -drive file=floppy.img,format=raw,if=floppy
```

qemuの画面で日時入力を各1回enterしてから…

```cmd
A>path a:\dos

A>mem /debug | more
...
A>
```

memコマンドの結果がページ単位に見れたらOKです。一応起動フロッピーディスクイメージが完成しました。qemuはまた閉じます。

## 5-2. HDDイメージにインストール

まずはイメージを作ります。ddでもいいんですが容量計算が面倒なので…

```sh
$ qemu-img create -f raw hdd.img 40M
```

ここでファイルシステムを作らずに、先のフロッピーも付けてそのままAドライブ(フロッピー)から起動します。

```sh
$ qemu-system-i386 -drive file=floppy.img,format=raw,if=floppy -drive file=hdd.img,format=raw -boot a
```

qemuの画面から

```cmd
A>path a:\dos
A>fdisk
```

1. メニューから`1`の`Create DOS Partition or Logical DOS Drive`を選択
1. メニューから`1`の`Create Primary DOS Partition`を選択
1. partitionをactiveにするか聞かれるので`y`を選択

これで一旦リブートされます。

```cmd
A>path a:\dos
A>format c: /s
```

1. フォーマットを続けるか聞かれるので、`y`
2. ボリュームラベルを聞かれるので、Enter

あとはフロッピーから適当にコマンドをコピーすれば起動可能なHDDが完成します。

```cmd
A>c:
C>mkdir dos
C>copy a:\dos\*.* c:\dos
...
C>
```

一旦QEMUを閉じて、今度はCドライブから起動します。

```sh
$ qemu-system-i386 -drive file=hdd.img,format=raw
```

起動が確認出来たら一旦完了です。MS-DOSの世界をご堪能ください。
次節に進むときはQEMUを閉じてください。

## 5-3. せっかくなのでMS-CとMASMも入れておく

ここからはオマケです。MS-DOS自体はもう起動するので…

まずは以下2つのファイルを用意します(**改行コードはCRLFにしてください**)。

```ini:config.sys
FILES=20
BUFFERS=20
SHELL=C:\COMMAND.COM C:\ /E:1024 /P
```

```bat:autoexec.bat
@echo off
set CL=
set LINK=
set MASM=
set COUNTRY=usa-ms
set BAKROOT=C:
set LIB=%BAKROOT%\tools\bld\lib
set INIT=%BAKROOT%\tools
set INCLUDE=%BAKROOT%\tools\bld\inc
set PATH=%BAKROOT%\dos;%BAKROOT%\tools
```

ではHDDイメージのパーティション情報から対象パーティションのオフセットを計算して、そこに`MS-DOS/v4.0/src/TOOLS`を突っ込み、上で用意した`config.sys`、`autoexec.bat`も放り込んでおきます。

```sh
$ fdisk -l hdd.img
ディスク hdd.img: 40 MiB, 41943040 バイト, 81920 セクタ
単位: セクタ (1 * 512 = 512 バイト)
セクタサイズ (論理 / 物理): 512 バイト / 512 バイト
I/O サイズ (最小 / 推奨): 512 バイト / 512 バイト
ディスクラベルのタイプ: dos
ディスク識別子: 0x00000000

デバイス   起動 開始位置 終了位置 セクタ サイズ Id タイプ
hdd.img1   *          63    80639  80577  39.3M  6 FAT16

$ echo $((63 * 512))
32256

$ mcopy -s -i hdd.img@@32256 MS-DOS/v4.0/src/TOOLS ::/
$ mcopy -i hdd.img@@32256 config.sys autoexec.bat ::/
```

エディタがないので、確認用Cコードを書いておきます(**改行コードはCRLFにしてください**)。

```c:hello.c
#include <stdio.h>

int main(int argc, char* argv[]) {
    printf("Hello, World!\n");
    return 0;
}
```

これも放り込んで、qemuを起動します。

```sh
$ mcopy -i hdd.img@@32256 hello.c ::/
$ qemu-system-i386 -drive file=hdd.img,format=raw
```

では試しにコンパイルしてみると…

```cmd
C>cl hello.c
...
C>hello
Hello, World!
C>
```

出来ました。QEMUを閉じてください。

## 5-4. ファイラとエディタを追加する

ここはさらにおまけで、往年のソフトをバイナリですが追加してみます。

まずは以下をDL。

- [LHAの詳細情報 : Vector ソフトを探す！](https://www.vector.co.jp/soft/dl/dos/util/se002413.html)^[LHAはソースからビルドも考えたのですが、LSI-Cが必要なのでやめました。せっかくMS-Cがあるので。差分はさらにバイナリを必要とするため、2.55だけを使っています。]
- [FD for IBM-PCの詳細情報 : Vector ソフトを探す！](https://www.vector.co.jp/soft/dos/util/se000160.html)^[日本語表示できないので英語版です]

エディタのVZだけはgitリポジトリがあるので、cloneして…

```sh
$ git clone https://github.com/vcraftjp/VZEditor.git
```

とりあえずHDDイメージ放り込んでQEMUを起動します。

```sh
$ mmd -i hdd.img@@32256 ::ARC
$ mcopy -i hdd.img@@32256 lha255.exe fdpc_313.lzh ::/ARC
$ mcopy -s -i hdd.img@@32256 VZEditor/SRC ::/ARC/VZSRC
$ mcopy -s -i hdd.img@@32256 VZEditor/VZ-IBM ::/ARC/VZ
$ qemu-system-i386 -drive file=hdd.img,format=raw
```

まずはlha.exe(2.55)を作ってtoolsに放り込みます。

```cmd
C>cd arc
C>md lha
C>cd lha
C>..\lha255
...
...[Y/N] Y
...
C>copy lha.exe c:\tools
C>cd ..
```

次にfd(fdpc)

```cmd
C>md fd
C>cd fd
C>lha x ..\fdpc_313.lzh
C>copy *.com c:\tools
C>copy *.cfg c:\tools
C>cd ..
```

vzはソースも付いててビルドも出来た^[.batや.makの修正が必要で、未解決シンボルあるけど、vzibm.comは出来るというだけ]のですが、それだけでは動かず、結局`vz-ibm`(HDD上ディレクトリは`vz`にしてある)が必要になるので、これもそのままバイナリを使います。

```cmd
C>cd vz
C>copy us\vzus.com c:\tools\vz.com
C>copy vzibm.def c:\tools\vz.def
C>zcopy @us\instus.lst c:\tools
C>cd ..
```

内容的にはinstall.batを動かしたときと同じことをしています。install.batは別ドライブからコピーする前提で実装されているので、同じドライブだと動きません。機種判定はIBM-PC判定のときの動作になります。

以上で、ファイラーとエディタの追加も完了です。こんな感じになります。

![fdとvzの使用風景](/images/deedff6243e27b/qemu_msdos4.webp)

# 6. まとめ

- MSが公開していたMS-DOS 4.0をビルドしてQEMU上で動かすことができた(日本語の表示と入力は不可)

# あとがき

MS-DOSを使っていた当時はPC-9801だったので、そういやDOS/Vで日本語MS-DOSって使ったことないなと気付かされました。