---
title: "Yocto 簡単に自前のC言語プログラムを追加する"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["raspberrypi", "yocto", "bitbake", "docker"]
publication_name: "singularity"
published: true
---
組込みエンジニア katsu です。

[前回の記事](https://zenn.dev/singularity/articles/20230628_raspi-for-aws-2)では、RaspberryPi向けにYoctoビルド環境の構築方法を記載しました。
このときには、Yoctoというビルドシステムを使って、自分に必要なOSSを組み込む基礎技術を学ぶ事ができました。

ただ、やっぱり自分で作ったプログラムも追加したいですよね。
ここでは、できるだけ手を抜きつつ、C言語で書いた自分のプログラム（HelloWorld）を組み込む方法を書いていきたいと思います。
もし、手っ取り早く結論だけ読みたい方は[こちら](#手っ取り早く組込みたい)を参照ください。

## 自分でレシピを作る
Yoctoビルドシステムの特徴として、個々のプログラムはビルド＆組み込み方法をレシピというファイルに記述し、複数のレシピを用途毎や特定ハードウェア単位にグルーピング化、階層化して、必要なプログラムをビルドしていきます。
このグルーピング化されたものをレイヤー（ここではBSPレイヤー）といい、"meta-xxx"フォルダで管理していきます。
ここでは、自分でBSPレイヤーを作成し、自前のC言語プログラムを追加します。

### 準備
環境は、[Yoctoビルド環境の構築方法](https://zenn.dev/singularity/articles/20230628_raspi-for-aws-2)を前提とします。
準備として、bitbakeコマンドを使うので、"oe-init-build-env"しましょう。
```
$ cd poky
$ source oe-init-build-env 

### Shell environment set up for builds. ###

You can now run 'bitbake <target>'

Common targets are:
    core-image-minimal
    core-image-full-cmdline
    core-image-sato
    core-image-weston
    meta-toolchain
    meta-ide-support

You can also run generated qemu images with a command like 'runqemu qemux86-64'.

Other commonly useful commands are:
 - 'devtool' and 'recipetool' handle common recipe tasks
 - 'bitbake-layers' handles common layer tasks
 - 'oe-pkgdata-util' handles common target package tasks
```

### BSPレイヤーを追加する
oe-init-build-envを実行すると、buildディレクトリに移動してしまうので、１つ上のフォルダに戻ります。
そして、meta-myapp レイヤーを作ります。
```
$ cd ..
$ bitbake-layers create-layer meta-myapp
NOTE: Starting bitbake server...
Add your new layer with 'bitbake-layers add-layer meta-myapp'
```

最終行にBSPレイヤーをYoctoに追加するコマンドが表示されています。
このコマンドをそのまま実行して、BSPレイヤーを追加します。
```
$ bitbake-layers add-layer meta-myapp
NOTE: Starting bitbake server...
```
以上で、ビルド対象に自分のレイヤーが追加されました。

・・・とはいえ、何が組み込まれたのでしょうか。
追加した meta-app フォルダ構成を見てみましょう。
```
$ tree meta-myapp
meta-myapp
|-- COPYING.MIT
|-- README
|-- conf
|   `-- layer.conf
`-- recipes-example
    `-- example
        `-- example_0.1.bb
```

設定ファイルっぽいlayer.conf を見てみましょう。
```bash:conf/layers.conf
# We have a conf and classes directory, add to BBPATH
BBPATH .= ":${LAYERDIR}"

# We have recipes-* directories, add to BBFILES
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "meta-myapp"
BBFILE_PATTERN_meta-myapp = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-myapp = "6"

LAYERDEPENDS_meta-myapp = "core"
LAYERSERIES_COMPAT_meta-myapp = "mickledore"
```
BBFILES が追加するレシピ（.bbファイル）のフォルダ構成が記載してあります。
recipes-xxxフォルダの下に、xxxフォルダ作って、その下にxxxのレシピファイルを追加しなさい、ということが書いてあります。
そう思ってみると、以下のフォルダ構成は納得ですね。
```
`-- recipes-example
    `-- example
        `-- example_0.1.bb
```
レシピファイルは、名前の後にバージョン番号をつけて、拡張子を.bbとします。
example_0.1.bbファイルの中身を見てみましょう。
``` bash: recipes-example/example/example_0.1.bb
SUMMARY = "bitbake-layers recipe"
DESCRIPTION = "Recipe created by bitbake-layers"
LICENSE = "MIT"

python do_display_banner() {
    bb.plain("***********************************************");
    bb.plain("*                                             *");
    bb.plain("*  Example recipe created by bitbake-layers   *");
    bb.plain("*                                             *");
    bb.plain("***********************************************");
}
```
SUMMARY, DESCRIPTION, LICENSEの記載が必要そうですね。
do_display_banner() は、単に処理が走ったというのを表示しているだけの処理です。

なお、layer.confにある.bbappend というファイルは、別のところにあるレシピを追加で修正したいときに使用するレシピです。
例えば、別BSPにあるプログラムにパッチを当てたいようなときに使用します。

### 自分のプログラムを追加する
では、自分のHelloWorldプログラムを作っていきましょう。

### レシピを追加する
先ほど解説で見たrecipes-exampleは邪魔なので、削除します。
```
cd meta-myapp
$ rm -fr receipes-example
```

続いて、myapp レシピを作ります。
layer.conf記載のフォルダ構成に合わせてフォルダ作成して、myapp_1.0.bbを作りましょう。
```
$ mkdir -p recipes-myapp/myapp
$ cd recipes-myapp/myapp
＄ touch myapp_1.0.bb
```

さて、C言語コードはどこにおきましょうか。
カッコよく書くなら、Yoctoシステムとは切り離して、別のリポジトリで管理する方が良いと思います。
ですが、簡単にするため、今回は、このBSPレイヤーの中に置くことにしましょう。
filesフォルダを作って、その中にHelloWorldプログラムと、Makefileを書きます。

```
$ mkdir files
```
HelloWorldプログラムはこちらです。
```c:main.c
#include <stdio.h>

int main(int argc, char *argv[])
{
	printf("Hello myapp world!\n");

	return 0;
}
```

Makefileはこちらです。
make すると myapp という実行ファイルが作られ,make clean すると myapp を削除します。
``` makefile:Makefile
TARGET=myapp
SRCS=main.c

all:
	${CC} ${SRCS} -o ${TARGET}

clean:
	rm -f {TARGET}
```

では、これをビルドしてくれるレシピを作成しましょう。

```bash:myapp_1.0.bb
SUMMARY = "My Application"
DESCRIPTION = "My Application in C."
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=1135ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://."

S = "${WORKDIR}"

TARGET_CC_ARCH += "${LDFLAGS}"

do_install() {
	install -d ${D}${bindir}
	install -m 0755 myapp ${D}${bindir}
}
```

| 変数 | 内容 |
| ---- | ---- |
| SUMMARY | 任意の文字列でOKです。”My Application"とします。 |
| DESCRIPTION | 任意の文字列でOKです。”My Application in C"とします。 |
| LICENSE | サンプルと同じMIT Licenseとします<br>他のライセンスでも、自前のライセンスでも良いと思います。 |
| LIC_FILES_CHKSUM | exampleのレシピにはありませんでしたが、組み込むライセンスファイルのmd5sum値を指定します。<br>ひとまずこのままお使いください。<br>ビルド時にエラーとなりますが、正解を教えてもらえます。 |
| SRC_URI | ソースコードのパスを指定します。<br>レシピと同一フォルダなので"file://."となります。<br>外部のgithubのリポジトリであれば、ここにgithub上のURLを記載します。 |
| S | ソースコードがあるフォルダを指定します。<br>WORKDIRとしておくと、使いそうなフォルダ名を調べてくれます。<br>filesフォルダを自動で見つけてくれます。 |
| TARGET_CC_ARCH | コンパイル時に、Yoctoビルドシステムで使うLDFLAGSを渡すようにします。<br>これを指定しないと、ビルド時にエラーとなります。 |
| do_install | 作成したプログラムをファイルシステム上のどこに置くかを指定します。<br>ここでは/bin下にmyappを置くようにします。 |

コンパイル時のCC変数やコンパイルオプションなどは、なんとYoctoビルドシステムのものが勝手に継承されてます。
do_compile() などで指定もできますが、特別なビルドが必要なければ、定義すら不要です。
こんな手抜きで、クロスコンパイルができるなんで、なんて幸せな世の中でしょう。

### ビルドする
buildフォルダに戻りましょう。
```
$ cd ../../../build
```

ビルド対象に、myapp を追加しましょう。
local.confに myapp　を追加します。
``` diff bash:conf/local.conf
@@ -282,6 +282,7 @@
 # this doesn't mean anything to you.
 CONF_VERSION = "2"
 
+IMAGE_INSTALL:append = " myapp"
```

ビルドします（失敗します）。
```
$ bitbake core-image-minimal
...

ERROR: myapp-1.0-r0 do_populate_lic: QA Issue: myapp: The LIC_FILES_CHKSUM does not match for file:///home/hoge/raspi2/poky/meta/files/common-licenses/MIT;md5=1835ade698e0bcf8506ecda2f7b4f302
myapp: The new md5 checksum is 0835ade698e0bcf8506ecda2f7b4f302
...
```
実行環境に含まれるMITライセンスファイルのハッシュ値を教えてくれるので、この値をレシピで指定しましょう.

``` diff bash:myapp_1.0.bb
@@ -1,7 +1,7 @@
 SUMMARY = "My Application"
 DESCRIPTION = "My Application in C."
 LICENSE = "MIT"
-LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=1135ade698e0bcf8506ecda2f7b4f302"
+LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"
```

再度ビルドします。
```
$ bitbake core-image-minimal
Loading cache: 100% |#############################################################| Time: 0:00:00
Loaded 1839 entries from dependency cache.
Parsing recipes: 100% |###########################################################| Time: 0:00:00
Parsing of 936 .bb files complete (935 cached, 1 parsed). 1839 targets, 74 skipped, 0 masked, 0 errors.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "2.4.0"
BUILD_SYS            = "aarch64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "arm-poky-linux-gnueabi"
MACHINE              = "raspberrypi2"
DISTRO               = "poky"
DISTRO_VERSION       = "4.2"
TUNE_FEATURES        = "arm vfp cortexa7 neon vfpv4 thumb callconvention-hard"
TARGET_FPU           = "hard"
meta                 
meta-poky            
meta-yocto-bsp       = "master:13b646c0e167ca52f69c91be5538049b172015ac"
meta-raspberrypi     = "master:dff85b9a9f66002856b9ed3b1aa3a384c0bc43d9"
meta-myapp           = "main:13c90ac1351528251fe7538ac3c01d897a0e655f"

Initialising tasks: 100% |#########################################################| Time: 0:00:01
Sstate summary: Wanted 375 Local 371 Mirrors 0 Missed 4 Current 1131 (98% match, 99% complete)
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 3459 tasks of which 3450 didn't need to be rerun and all succeeded.
```
今度は成功しました！

RaspberryPiのイメージファイルができているか確認します。
```
ls tmp/deploy/images/raspberrypi2/*.wic.bz2
tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2-20231013115858.rootfs.wic.bz2
tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2.wic.bz2
```

### RaspberryPiにイメージファイルを焼く
[前回](https://zenn.dev/singularity/articles/20230628_raspi-for-aws-2)と同様、
Mac側のターミナルから、"docker cp" コマンドでイメージファイルをとってきます（解凍します）。
```
$docker ps
CONTAINER ID   IMAGE          COMMAND       CREATED       STATUS       PORTS     NAMES
b7c4cb3eeb5c   ubuntu:22.04   "/bin/bash"   3 weeks ago   Up 2 weeks             raspi2-env
$
$ docker cp b7c4cb3eeb5c:/home/hoge/raspi2/poky/build/tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2-20231013115858.rootfs.wic.bz2 .
$
$ bunzip2 core-image-minimal-raspberrypi2-20231013115858.rootfs.wic.bz2
```

書き込みには RaspberryPi Imager を使用しましょう。
RaspberryPi Imagerを起動します。
![](/images/20230611_010.png)

書き込むOSを選択します。
一番下の「カスタムイメージを使う」から、先ほど用意したビルドイメージを選択します。
![](/images/20230628_210.png)

RaspberyPi用のSDカードをMacに挿し、ストレージに選択します。
![](/images/20230819_010.png)

「書き込む」ボタンを押して、しばらく待ってください。
10秒ぐらいで完了しました。
![](/images/20230819_020.png)
完了したら、そのままSDカードを抜いてOKです。

### RaspberryPiでmyappを実行する
作成したイメージには ssh で、IPアドレス直指定（ここでは192.168.1.10とします）で接続しましょう（IPアドレスがわからない時は、ディスプレイ・キーボードをRaspberryPiに接続してキーボードから入力してください）。
Mac側のターミナルから、rootアカウントで接続してください。
```
＄ ssh root@192.168.1.10
```

myappコマンドが組み込まれているか、確認します。
```
# /bin/myapp 
Hello myapp world!
```
ちゃんと指定した/binいかにmyappファイルがあり、実行できました。

ということで、自分でBSPレイヤーを作って、自前のC言語プログラムをRaspberryPiで実行することができました。

## トラブルシュート
改めて、ビルド（bitbake core-image-minimal）時のエラーで、はまりポイント記載しておきます。

### LIC_FILES_CHKSUM に記載するチェックサム値がわからない？
ライセンスファイルのエラーが表示されますが、
エラーの最終行に、正解が書いてあるので、レシピに記載してください。
```
ERROR: myapp-1.0-r0 do_populate_lic: QA Issue: myapp: LIC_FILES_CHKSUM is not specified for file:///home/hoge/raspi2/poky/meta/files/common-licenses/MIT;md5=
myapp: The md5 checksum is 0835ade698e0bcf8506ecda2f7b4f302 [license-checksum]
```

### LDFLAGSが指定されていない？
レシピでTARGET_CC_ARCHを指定しなかった場合は、以下のようなエラーが出ます。
LDFLAGSを指定してください。
```
ERROR: myapp-1.0-r0 do_package_qa: QA Issue: File /usr/bin/myapp in package myapp doesn't have GNU_HASH (didn't pass LDFLAGS?) [ldflags]
```

### C言語のソースコードが見つからない？
ソースコードのディレクトリを files としましたが、別のフォルダ名でやろうとすると参照できなくて以下のようなエラーとなります。
エラー表示に、サーチしてくれるフォルダパスが記載されているので、そこに格納してください。
今回は一番最後のfilesとしました。
```
ERROR: /home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp_1.0.bb: Unable to get checksum for myapp SRC_URI entry .: file could not be found
The following paths were searched:
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp-1.0/poky/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp/poky/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/files/poky/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp-1.0/raspberrypi2/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp/raspberrypi2/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/files/raspberrypi2/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp-1.0/armv7ve/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp/armv7ve/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/files/armv7ve/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp-1.0/rpi/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp/rpi/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/files/rpi/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp-1.0/arm/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp/arm/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/files/arm/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp-1.0/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/myapp/.
/home/hoge/raspi2/poky/meta-myapp/recipes-myapp/myapp/files/.
ERROR: Parsing halted due to errors, see error messages above
```

## 手っ取り早く組込みたい
レシピとかC言語のファイルを自分で準備するのが面倒。まずはここに書いてあるmyappを組み込んで、自分のプログラムのベースにしたいという方
meta-myappを持っていってください。
https://github.com/ka2yan/meta-myapp


### 準備
まずは、Yoctoビルドシステムのトップに移動して、oe-init-build-env を実行してbitbakeコマンドが使えるようにします。
```
$ cd poky
$ source oe-init-build-env
```

build ディレクトリに移動してしまうので、トップディレクトリに戻っておきます。
```
cd ..
```

### myapp をビルド対象に組み込む
meta-myapp を git cloneします。
```
$ git clone https://github.com/ka2yan/meta-myapp
```

meta-myapp レイヤーをビルドシステムに追加します。
```
$ bitbake-layers add-layer meta-myapp
```

build/conf/local.conf に、”myapp" を追加してビルド対象にします。
```diff bash:build/conf/local.conf
@@ -281,3 +281,15 @@
 # track the version of this file when it was generated. This can safely be ignored if
 # this doesn't mean anything to you.
 CONF_VERSION = "2"
+
+IMAGE_INSTALL:append = " myapp"
"
```

### ビルドする
oe-init-build-env して、bitbakeコマンドでビルドしましょう。
```
$ cd build
$ bitbake core-image-minimal
Loading cache: 100% |###################################################| Time: 0:00:00
Loaded 4508 entries from dependency cache.
Parsing recipes: 100% |#################################################| Time: 0:00:00
Parsing of 2752 .bb files complete (2751 cached, 1 parsed). 4508 targets, 171 skipped, 0 masked, 0 errors.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "2.4.0"
BUILD_SYS            = "aarch64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "arm-poky-linux-gnueabi"
MACHINE              = "raspberrypi2"
DISTRO               = "poky"
DISTRO_VERSION       = "4.2"
TUNE_FEATURES        = "arm vfp cortexa7 neon vfpv4 thumb callconvention-hard"
TARGET_FPU           = "hard"
meta                 
meta-poky            
meta-yocto-bsp       = "master:13b646c0e167ca52f69c91be5538049b172015ac"
meta-raspberrypi     = "master:dff85b9a9f66002856b9ed3b1aa3a384c0bc43d9"
meta-myapp           = "master:13b646c0e167ca52f69c91be5538049b172015ac"

Initialising tasks: 100% |#############################################| Time: 0:00:01
Sstate summary: Wanted 406 Local 394 Mirrors 0 Missed 12 Current 1251 (97% match, 99% complete)
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 3808 tasks of which 3786 didn't need to be rerun and all succeeded.
```
ビルドエラーした場合は、[トラブルシュート](#トラブルシュート)を参照ください。

どうでしょうか。
あとは、meta-myapp/myapp/myapp/files下にあなたのプログラムを置いてRaspberryPiで動かしてみましょう。

:::message
毎回全ビルドが面倒
毎回イメージまで作ると時間がかかるので、以下のコマンドで特定プログラムのみビルドすることができます。

myappのみをビルドする
```
$ bitbake myapp
```

myappビルド結果を削除する
```
$ bitbake -c cleansstate myapp
```
:::


## 参考
大変助かりました。ありがとうございました。

https://mcommit.hatenadiary.com/entry/2019/12/05/235100#bitbakeがレイヤとレシピを認識する流れ

https://qiita.com/propella/items/8d861d08c95ac08136d2


