---
title: "Yocto 簡単に自前のOSSパッチを組み込む"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["raspberrypi", "yocto", "bitbake", "docker"]
publication_name: "singularity"
published: true
---
[前回の記事](https://zenn.dev/singularity/articles/20230628_raspi-yocto)では、RaspberryPi向けにYoctoビルド環境に自分で作ったプログラムを追加しました。
ただ、実際にものを作るときには、必ずしも自分のプログラムだけではなく、必要に応じてOSSを自分で修正する必要があります。
という訳で、今回は、世の中に公開されているOSSに自分パッチを当てる方法を書いていきます。

## 今回作るもの
今回は、curlコマンドに自分で作ったパッチを適用して、RaspberryPi向けのイメージを作成します。
具体的には、BSPレイヤー"meta-mycmd" を作って、その中でcurlコマンドの追加レシピとパッチファイルを格納します。
これにより、パッチが適用されたcurlコマンド入りのRaspberryPiイメージがビルドできるようになります。

## 準備
環境は、[Yoctoビルド環境の構築方法](https://zenn.dev/singularity/articles/20230628_raspi-for-aws-2)を前提とします。
ではbitbakeコマンドを使えるように、"oe-init-build-env"しましょう。
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

## BSPレイヤーを追加する
oe-init-build-envを実行すると、buildディレクトリに移動してしまうので、１つ上のディレクトリに戻ります。
そして、meta-mycmd レイヤーを作ります。
```
$ cd ..
$ bitbake-layers create-layer meta-mycmd
NOTE: Starting bitbake server...
Add your new layer with 'bitbake-layers add-layer meta-mycmd'
```

最終行にBSPレイヤーをYoctoに追加するコマンドが表示されています。
このコマンドをそのまま実行して、BSPレイヤーを追加します。
```
$ bitbake-layers add-layer meta-mycmd
NOTE: Starting bitbake server...
```
以上で、ビルド対象に自分のレイヤーが追加されました。

## パッチファイルを作る
今回パッチを当てる対象のcurlのバージョンを調べます。
```
$ bitbake-layers show-recipes | grep -A 1 curl
curl:
  meta                 8.1.2
```
curlは、meta レイヤーにレシピがあって、バージョンは 8.1.2 であることを示しています。

curl 8.1.2のコードを持ってきて、修正を入れましょう。
:::message
自分向けメモ
手抜きパッチで良ければ、以下のような感じで取ってきて、修正できます。
$ git clone https://github.com/curl/curl.git -b curl-8_1_2
$ cd curl/src
$ vi ...
:::
今回は、お試しとしてコマンドを起動したときに、”I am RaspberryPi"を最初に出力するパッチを作ります。

git diff した結果が以下です。
```diff
diff --git a/src/tool_main.c b/src/tool_main.c
index 2b7743a7e..dabad3903 100644
--- a/src/tool_main.c
+++ b/src/tool_main.c
@@ -274,6 +274,7 @@ int main(int argc, char *argv[])
 #if defined(HAVE_SIGNAL) && defined(SIGPIPE)
   (void)signal(SIGPIPE, SIG_IGN);
 #endif
+  fprintf(stdout, "I am RaspberryPi\n");
 
   /* Initialize memory tracking */
   memory_tracking_init();
```

最近までは、このdiffをそのままファイルに落としてYoctoに組み込めたのですが、
Yocto４.0から、パッチファイルの位置付けを明記しなければならなくなりました。
（今のところ、metaとか公式のBSPレイヤーへのパッチのみっぽい?）

先頭に、Upstream-Status の１行を追加してください。[ ]は理由を記載するところです。
最終的なパッチは以下になります。

```diff:0001-add-printf.patch
Upstream-Status: Inappropriate [this is test code]
diff --git a/src/tool_main.c b/src/tool_main.c
index 2b7743a7e..dabad3903 100644
--- a/src/tool_main.c
+++ b/src/tool_main.c
@@ -274,6 +274,7 @@ int main(int argc, char *argv[])
 #if defined(HAVE_SIGNAL) && defined(SIGPIPE)
   (void)signal(SIGPIPE, SIG_IGN);
 #endif
+  fprintf(stdout, "I am RaspberryPi\n");
 
   /* Initialize memory tracking */
   memory_tracking_init();
```
:::message
Upstream-Status の記載できるStatusは以下。
| Status | 意味 |
| ---- | ---- |
| Pending | まだ決定が下されていないか、まだアップストリームに提出されていない |
| Submitted | アップストリームに提出され、承認を待っている |
| Accepted | アップストリームで受け入れられ、次のアップデートで削除されると予想されます。期待されるバージョン情報を含めてください |
| Backport | 新しいアップストリームバージョンからのバックポート。特定のバージョンに固定されているため、アップストリームバージョン情報を含めてください |
| Denied | アップストリームに受け入れられていません。パッチの理由を含めてください |
| Inappropriate[reason] | パッチがアップストリームに適していない。同じ行に[]で囲まれた簡潔な理由を含めてください |

詳細は[Yoctoプロジェクトのメール](https://docs.yoctoproject.org/pipermail/yocto/2011-April/001237.html)を参照ください。
:::

## レシピを作る
meta-mycmd ディレクトリには、recipes-exampleがありますが使わないので、削除しておきましょう
```
$ cd meta-mycmd
$ rm -fr recipes-example
```

curlコマンドは meta レイヤーにあるので、metaディレクトリの構成を確認します。
```
$ tree meta
meta
|-- ...
`-- recipes-support
    |-- ...
    `-- curl
        |-- ...
        `-- curl_8.1.2.bb
```
recipes-support/curl ディレクトリの下に、curlコマンドのレシピ（curl_8.1.2.bb）があります。

わかりやすくするため、meta-mycmd ディレクトリの構成も合わしましょう。
以下のように追加レシピ（.bbappend）と、パッチファイルを置くようにします。
```
meta-mycmd
|- ...
`-- recipes-support
    `-- curl
        |-- curl_%.bbappend
        `-- files
            `-- 0001-add-printf.patch
```
追加レシピのファイル名は、curl_%.bbappendです。
'%'は、任意のバージョンを示しています。
ここでは、curl本体のバージョンが8.1.2以外でも、このパッチは適用されるように'%'にしています。
もし、特定のバージョンだけに適用したい場合は、'%'ではなく、curl_8.1.2.bbappendなどのようにバージョンを明記してください。

では、これに合わせてディレクトリを作って、.bbappendファイルを作りましょう。
```
$ mkdir -p recipes-support/curl
$ cd recipes-support/curl
$ vi curl_%.bbappend
```

.bbappendファイルには以下を記載ください。
```bash:curl_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI:append = " file://0001-add-printf.patch"
```
それぞれの変数の意味は以下になります。
| 変数 | 内容 |
| ---- | ---- |
| FILEEXTRAPATHS:prepend | 追加レシピで使用するファイル置き場のパス |
| SRC_URI:append | 差分コード記載されたパッチファイル |

最後に、先ほど作ったパッチファイルを 0001-add-printf.patchという名前で、filesディレクトリに置いてください。

:::message
- Yocto 3.4未満の書き方
今回は、Yocto 4.2 なんですが、Yocto.３.4未満の方は以下になります（たぶん）。
[本家のサイト](https://docs.yoctoproject.org/migration-guides/migration-3.4.html)に書き方のサンプル（Before／After）がありますので、ご参考にしてください。
```bash:curl_%.bbappend（Yocto３.4未満）
FILESEXTRAPATHS_prepend := "${THISDIR}/files:"

SRC_URI ＋= " file://0001-add-printf.patch"
```

- アーキテクチャ毎のパッチ作成
他の.bbappendファイルを見ると、以下のような書き方をしているものがあります。
３つ目のqemux86, qemuarm,rpiですが、これはCPU/BSPなどのアーキテクチャを表しています。
例えば、qemux86とか、rpi(RaspberryPi)向けにはこのパッチを使え、というものですね。
```bash:hogehoge.bbappend
SRC_URI:append:qemux86 = " file://somefile1"
SRC_URI:prepend:qemuarm = "file://somefile2"
SRC_URI:prepend:rpi = "file://somefile3"
```

- レシピのバージョン指定
多くのレシピでは、hogehoge_1.0.bb などバージョンが入っています。
このようなバージョン入りのレシピに.bbappendで追加指定するときは、参照するレシピ（.bb）のバージョンに合わせるか、任意のバージョン（％）であることを明示する必要があります。
もし、元のレシピ（.bb）にバージョンがない場合は、バージョンなしの追加レシピ（.bbappend）としています。
例えば、curl.bbの場合は、curl.bbappendとなります。
ここを合わせないと正しく認識されません。
:::


## 追加したレシピ・パッチを確認する
確認には、bitbake-layerコマンドが大活躍します。
まずは、show-layersで、meta-awsappのBSPレイヤーが正しく認識されいているのか確認しましょう。
```
$ bitbake-layers show-layers

NOTE: Starting bitbake server...
layer                 path                                               priority
=================================================================================
core                  /home/hoge/raspi2/poky/meta                        5
yocto                 /home/hoge/raspi2/poky/meta-poky                   5
yoctobsp              /home/hoge/raspi2/poky/meta-yocto-bsp              5
raspberrypi           /home/hoge/raspi2/poky/meta-raspberrypi            9
meta-mycmd            /home/hoge/raspi2/poky/meta-mycmd                  6
```
meta-mycmdが認識されています。
パッチ適用の順番は、priorityと順番によって決まります。
数字が小さい方が優先度が高く、数字の大きい方が優先度が低くなります。
同じ番号の場合は上に書いてある方が優先されます。

続いて show-appendsで、追加レシピ（.bbappend）が正しく認識されているのか確認しましょう。
```
$ bitbake-layers show-appends

NOTE: Starting bitbake server...
Loading cache: 100% |################################################| Time: 0:00:00
Loaded 1838 entries from dependency cache.
=== Appended recipes ===
...
curl_8.1.2.bb:
  /home/hoge/raspi2/poky/meta-mycmd/recipes-support/curl/curl_%.bbappend
...
```
curl_8.1.2.bbの追加レシピとして、curl_%.bbappendファイルを認識していることがわかります。


:::message
改めてYoctoでパッチ追加時に使用するコマンドを記載しておきます。

BSPレイヤーを確認する
```
bitbake-layers show-layers
```

組込み可能なレシピ名を確認する
build/conf/loca.confで組み込むプログラム名です。
```
bitbake-layers show-recipes
```

適用されるパッチとその順番を確認する
```
bitbake-layers show-appends
```
:::

## ビルドする
buildフォルダに戻りましょう。
```
$ cd ../../../build
```

ビルド対象（local.conf）に curlコマンドが入っているのを確認します。
``` diff bash:conf/local.conf
@@ -282,6 +282,7 @@
 # this doesn't mean anything to you.
 CONF_VERSION = "2"
 
+IMAGE_INSTALL:append = " curl"
```

ビルドします。
```
$ bitbake core-image-minimal
Loading cache: 100% |                                                  | ETA:  --:--:--
Loaded 0 entries from dependency cache.
Parsing recipes: 100% |################################################| Time: 0:00:39
Parsing of 935 .bb files complete (0 cached, 935 parsed). 1838 targets, 74 skipped, 0 masked, 0 errors.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "2.4.0"
BUILD_SYS            = "aarch64-linux"
NATIVELSBSTRING      = "ubuntu-22.04"
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
meta-mycmd           = "master:13b646c0e167ca52f69c91be5538049b172015ac"

Initialising tasks: 100% |############################################| Time: 0:00:01
Sstate summary: Wanted 1498 Local 0 Mirrors 0 Missed 1498 Current 0 (0% match, 0% complete)
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 3443 tasks of which 6 didn't need to be rerun and all succeeded.
```
無事終了しました。

バイナリイメージを確認します。
```
$ ls -lh tmp/deploy/images/raspberrypi2/*.wic.bz2
-rw-r--r-- 2 hoge hoge 28M Dec  2 22:10 tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2-20231202103145.rootfs.wic.bz2
lrwxrwxrwx 2 hoge hoge  61 Dec  2 22:10 tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2.wic.bz2 -> core-image-minimal-raspberrypi2-20231202103145.rootfs.wic.bz2
```
28MBのイメージファイルができました。

本当にパッチ当たってるのかなー、っていう心配性なあなた。
ビルドされたコードを見てみましょう。
RaspberryPi向けのコードは以下にあります。gitフォルダ以下にパッチが当たったコードがあるはずです。
```
$ cat tmp/work/cortexa7t2hf-neon-vfpv4-poky-linux-gnueabi/curl/8.1.2-r0/curl-8.1.2/src/tool_main.c 
```

:::message
パッチがうまく当たらずエラーして、修正後に再ビルドしたいときは以下のbitbakeコマンドを使いましょう。
```
$ bitbake -c compile -f curl
```
:::

## 動作確認する
RaspberryPiに焼いて、実際の動作確認してみてください。
Raspberry Pi Imagerで生成したイメージをSDカードに焼いて、起動します。
[前回の記事](https://zenn.dev/singularity/articles/20230820_raspi-for-aws-3#raspberrypiからメッセージを送信する%E3%80%82)を参照ください。



## トラブルシュート
今回も色々とハマってしまいました。
注意点を記載しておきます。

### 1) ディレクトリ名、ファイル名を間違えるな
私は実際タイポしました。間違えないようにしましょう。
間違えても無視されるだけで、エラーが出ず、わかりにくいです。

|  正解  |  間違い |
| ---- | ---- |
| recipes | receipes |
| .bbappend | .bbapend |

### 2) 追加レシピのファイル名を間違えるな
参照するレシピ（.bb）にバージョンがある場合とない場合があります。
参照するレシピに合わせた追加レシピ（.bbappend）を作りましょう。

|  バージョンがある場合  |  バージョンがない |
| ---- | ---- |
| name_%.bbappend | name.bbappend |

ファイル名を間違えた時のエラーは以下です。
```
＄  bitbake-layers show-appends
...
ERROR: No recipes in default available for:
  /home/hoge/raspi2/poky/meta-awsapp/recipes-support/curl/curl.bbappend
```

### 3) .bbappend 書き方を間違えるな
FILESEXTRAPATHS, SRC_URI がYoctoのバージョンによって書き方が違います。
検索などでは新旧情報が混ざってヒットするので、注意しましょう。

```bash:curl_%.bbappend（Yocto3.4以降）
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI:append = " file://0001-add-printf.patch"
```

```bash:curl_%.bbappend（Yocto３.4未満）
FILESEXTRAPATHS_prepend := "${THISDIR}/files:"

SRC_URI ＋= " file://0001-add-printf.patch"
```

### 4) 環境変数、do_compileの実際の処理内容を確認する
ビルド時の環境変数や、do_patch, do_compileで、何をしているのか知りたい時、以下のコマンドが役立ちました。
```
$ bitbake -e curl

```
### 5) meta-xx を別名に変更しない
bitbake-layer create-layerせずに、既にあるBSPレイヤーをcpコマンドなどでコピーしてrenameするのはやめた方が良いです。
内部の設定ファイル（conf/layer.conf）にレイヤー名が入っているので、この辺も一緒に変更が必要になります。

### 6) filesディレクトリの指定が間違っている
追加レシピ（.bbappend）のEILESEXTRAPATHS指定が正しくない場合は、以下のようなエラーが出ます。
```
ERROR: /home/hoge/raspi2/poky/meta/recipes-support/curl_8.1.2.bb: Unable to get checksum for nativesdk-curl SRC_URI entry 0001-add-print.patch: file could not be found
The following paths were searched:
```

### 7) Upstream-Statusがない
```
ERROR: curl-native-8.1.2-r0 do_patch: QA Issue: Missing Upstream-Status in patch
/home/hoge/raspi2/poky/meta-mycmd/recipes-support/curl/files/0001-add-printf.patch
Please add according to https://www.openembedded.org/wiki/Commit_Patch_Message_Guidelines#Patch_Header_Recommendations:_Upstream-Status . [patch-status]
```

Yocto 4.0から、適用するパッチの状態を明記が必要になりました。
git diffコマンドで出力したパッチファイルをそのまま適用できなくなり、Upstream-Status で、Upstream（本流）に適用すべきものなのかどうかを Pending, Submitted, Accepted, Backport, Denied, Inappropriate[reason]などで明記する必要が出てきました（それぞれの意味は上に記載しました）。

うーん、公式なビルドには良いですけど、ちょっと敷居が高くなりますね。。。
今回は、１行追加したやり方があまりかっこよくないので、スマートなやり方がありましたら教えてください。


## 参考
以下のページは、大変助かりました。ありがとうございました。

https://yoctobbq.lineo.co.jp/?q=node/385

https://docs.yoctoproject.org/migration-guides/migration-3.4.html

https://yoctobbq.lineo.co.jp/?q=node/433


