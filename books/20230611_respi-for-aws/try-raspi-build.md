---
title: "RaspberryPiをゼロからビルドする"
---

ここでは、Dockerを使ってUbuntu環境を作った後、
Yoctoを使ってRaspberryPi向けのビルドをゼロからやってみます。

## Yoctoとは
最近のLinuxベースの組込み機器は、Yoctoというビルドシステムを使ってビルドします。

Yoctoがやることは、組込み機器向けのコンパイラ（gcc)を作って、
OSのLinux kernelはもちろん、関連するコマンドや各種ライブラリの依存関係を考慮して
機器に組み込むべきOSSを全てダウンロードして、エラーしないような順番でビルドして、
機器にそのまま使用できるイメージまで作ってくれる優れものです。
x
一昔前は、この全てを手動でやっていました。
当然ものすごい時間が掛かっていたし、最後まで作るのは職人技でした。


## Yocto仕組み
Yoctoビルドシステムを図で表すと以下のようになります。
図
ちょっと難しそうですね。
とは言え、まずは知っておくべきなのは以下です。
### Bitbake
### Meta-Layer
### Recipe
### ビルドコマンド
### フォルダ構成

では、事前知識は得たので、早速やってみましょう。

## Docker インストール
Homebrewで簡単にお手軽インストールしましょう。
ターミナルから以下のコマンドでインストールします。
```
$ brew install docker
```
しばらく待って、完了です。

インストールできたか、以下で確認します。
```
$ docker --version
Docker version 23.0.5, build bc4487a
```
起動しておきます。
```
$ open /Applications/Docker.app
```
こんな画面になります。

画像１

とはいえ、ここでは、ターミナルで操作しますので、どこか見えないところに置いておきましょう。

## Ubuntu22.04環境を構築します
YoctoがサポートしているLinux 環境のうち、今回は Ubuntu22.04を使用します。
[Yocto公式ページ](https://docs.yoctoproject.org/ref-manual/system-requirements.html?highlight=ubuntu#supported-linux-distributions)

Macのターミナルから、DockerコマンドでUbuntuのイメージをとってきます。
```
$ docker pull ubuntu:22.04
```
イメージは、実体ではなく確か手順書だけな感じなので、すぐに完了します。

イメージが取得できていることを確認しましょう。
```
$ docker image ls
REPOSITORY              TAG       IMAGE ID       CREATED        SIZE
ubuntu                  22.04     3f5ef9003cef   1 days ago    69.2MB
```

コンテナ（実体）を作成します。
コンテナ名は、”raspi2-env" とします。以降、この名前を使用します。
```
$ docker run -it -d --name raspi2-env ubuntu:22.04
```

”raspi2-env"が、作成できたか確認します。
```
$ docker container ps
CONTAINER ID   IMAGE          COMMAND       CREATED          STATUS          PORTS     NAMES
b7c4cb3eeb5c   ubuntu:22.04   "/bin/bash"   48 seconds ago   Up 47 seconds             raspi2-env
```

では、コンテナの中に入ります。
```
$ docker exec -it raspi2-env /bin/bash
root@b7c4cb3eeb5c:/#  
```
無事入れました！
ここで実行するコマンドは、Ubuntuの中で実行するコマンドになります。
```
root@b7c4cb3eeb5c:/#  uname -a
Linux b7c4cb3eeb5c 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022 aarch64 aarch64 aarch64 GNU/Linux
root@b7c4cb3eeb5c:/# ls
bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@b7c4cb3eeb5c:/# 
```

:::message
ここで、一旦、Dockerコマンをまとめておきます。

コンテナ内での作業の開始
```
$ docker exec -it raspi2-env /bin/bash
```

コンテナ内での作業の終了
```
root@b7c4cb3eeb5c:/# exit
＄ 
```



コンテナの起動と停止コマンドは以下です。
コンテナ内の作業を完全に終了するなら落としておきましょう。

コンテナの開始
```
$ docker start raspi2-env
```
コンテナの停止
```
$ docker stop raspi2-env
```
:::

## 必要なコマンドをUbuntu環境にインストールします
まずは、各種パッケージを更新しておきます。
```
root@b7c4cb3eeb5c:/# apt update
root@b7c4cb3eeb5c:/# apt -y upgrade
```

よく使うコマンドは入れておきます。
エディタと、bashタブ補完するパッケージです。
```
apt install -y vim bash-completion
```

Yoctoで使用するパッケージを以下記載されています。
[Yocto公式ページ](https://docs.yoctoproject.org/ref-manual/system-requirements.html#required-packages-for-the-build-host)
これに従って、インストールします。
途中、タイムゾーン（tzdata）を選択させられるので、Asia(6), Tokyo(79)を選択します。

```
root@b7c4cb3eeb5c:/# apt install -y gawk wget git diffstat unzip texinfo gcc build-essential chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev python3-subunit mesa-common-dev zstd liblz4-tool file locales
...
Configuring tzdata
------------------

Please select the geographic area in which you live. Subsequent configuration questions will narrow this down by
presenting a list of cities, representing the time zones in which they are located.

  1. Africa   3. Antarctica  5. Arctic  7. Atlantic  9. Indian    11. US
  2. America  4. Australia   6. Asia    8. Europe    10. Pacific  12. Etc
Geographic area: 6
Please select the city or region corresponding to your time zone.

  1. Aden      14. Beirut      27. Gaza         40. Karachi       53. Muscat        66. Riyadh         79. Tokyo
  2. Almaty    15. Bishkek     28. Harbin       41. Kashgar       54. Nicosia       67. Sakhalin       80. Tomsk
  3. Amman     16. Brunei      29. Hebron       42. Kathmandu     55. Novokuznetsk  68. Samarkand      81. Ujung_Pandang
  4. Anadyr    17. Chita       30. Ho_Chi_Minh  43. Khandyga      56. Novosibirsk   69. Seoul          82. Ulaanbaatar
  5. Aqtau     18. Choibalsan  31. Hong_Kong    44. Kolkata       57. Omsk          70. Shanghai       83. Urumqi
  6. Aqtobe    19. Chongqing   32. Hovd         45. Krasnoyarsk   58. Oral          71. Singapore      84. Ust-Nera
  7. Ashgabat  20. Colombo     33. Irkutsk      46. Kuala_Lumpur  59. Phnom_Penh    72. Srednekolymsk  85. Vientiane
  8. Atyrau    21. Damascus    34. Istanbul     47. Kuching       60. Pontianak     73. Taipei         86. Vladivostok
  9. Baghdad   22. Dhaka       35. Jakarta      48. Kuwait        61. Pyongyang     74. Tashkent       87. Yakutsk
  10. Bahrain  23. Dili        36. Jayapura     49. Macau         62. Qatar         75. Tbilisi        88. Yangon
  11. Baku     24. Dubai       37. Jerusalem    50. Magadan       63. Qostanay      76. Tehran         89. Yekaterinburg
  12. Bangkok  25. Dushanbe    38. Kabul        51. Makassar      64. Qyzylorda     77. Tel_Aviv       90. Yerevan
  13. Barnaul  26. Famagusta   39. Kamchatka    52. Manila        65. Rangoon       78. Thimphu
Time zone: 79
...
done.
root@b7c4cb3eeb5c:/# 
```
数分待つと完了しました。

locale設定します。
```
root@b7c4cb3eeb5c:/# locale-gen en_US.UTF-8
```

ここで、最低限の準備が完了しました。

ですが、”root" のまま作業するのも気持ち悪いので、
ユーザー"hoge"を作って、
パスワード設定し、
そちらで作業をしましょう。

```
root@b7c4cb3eeb5c:/# useradd -m hoge
root@b7c4cb3eeb5c:/# passwd hoge
New password:
Retype new password:
passwd: password updated successfully
root@b7c4cb3eeb5c:/# su - hoge
$ 
```

今のアカウントを確認します。
```
$ whoami
hoge
```
”hoge"になりました。

## Yoctoビルドシステムを導入する
ここからYoctoを使って、RaspberryPiのビルド準備を行います。

### Yocto本体を取得する
作業ディレクトリを作って、Yocto本体を持ってきます。
git clone は、数分ほどかかります。
```
$ mkdir raspi2
$ cd raspi2
$ git clone git://git.yoctoproject.org/poky
```
中身を確認します。
```
$ ls poky
LICENSE		      MAINTAINERS.md  README.OE-Core.md   README.poky.md  contrib	 meta-poky	meta-yocto-bsp
LICENSE.GPL-2.0-only  MEMORIAM	      README.hardware.md  README.qemu.md  documentation  meta-selftest	oe-init-build-env
LICENSE.MIT	      Makefile	      README.md		  bitbake	  meta		 meta-skeleton	scripts
```
meta-xxx というのが、BSPレイヤーです。
現在は基本的なものしかありません。

### RaspberryPi BSPレイヤーを取得する
ここに、RaspberryPi向けのレイヤーを追加します。
```
$ cd poky
$ git clone git://git.yoctoproject.org/meta-raspberrypi
```

meta-raspberrypi というレイヤーが追加されました。
```
$ ls
LICENSE		      MEMORIAM		  README.md	  contrib	 meta-raspberrypi  oe-init-build-env
LICENSE.GPL-2.0-only  Makefile		  README.poky.md  documentation  meta-selftest	   scripts
LICENSE.MIT	      README.OE-Core.md   README.qemu.md  meta		 meta-skeleton
MAINTAINERS.md	      README.hardware.md  bitbake	  meta-poky	 meta-yocto-bsp
```
現時点では、meta-raspberrypiフォルダが展開されただけで、ビルド対象には組み込まれていません。

### ビルド環境を整える
ビルドに使用する環境変数などを設定してくれるスクリプトを実行します。
正しく実行できると、カレントディレクトリが "poky/build"に移動します。

```
$ source oe-init-build-env
You had no conf/local.conf file. This configuration file has therefore been
created for you from /home/hoge/raspi2/poky/meta-poky/conf/templates/default/local.conf.sample
You may wish to edit it to, for example, select a different MACHINE (target
hardware).

You had no conf/bblayers.conf file. This configuration file has therefore been
created for you from /home/hoge/raspi2/poky/meta-poky/conf/templates/default/bblayers.conf.sample
To add additional metadata layers into your configuration please add entries
to conf/bblayers.conf.

The Yocto Project has extensive documentation about OE including a reference
manual which can be found at:
    https://docs.yoctoproject.org

For more information about OpenEmbedded see the website:
    https://www.openembedded.org/


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
実行結果を読むと、以下のようなキーワードが出ています。
- conf/local.conf
- conf/bblayers.conf
- bitbake <target>

### RaspberryPi　BSPレイヤーを追加する
bblayers.conf に ”meta-raspberrypi" レイヤーを追加すると、
ビルド対象に追加することができます。
```diff bash:conf/bblayers.conf
@@ -9,4 +9,5 @@
   /home/hoge/raspi2/poky/meta \
   /home/hoge/raspi2/poky/meta-poky \
   /home/hoge/raspi2/poky/meta-yocto-bsp \
+  /home/hoge/raspi2/poky/meta-raspberrypi\
   "
```

### ビルド対象を、RaspberryPi2に設定する
```diff bash:conf/local.conf
@@ -36,7 +36,8 @@
 # This sets the default machine to be qemux86-64 if no other machine is selected:
-MACHINE ??= "qemux86-64"
+#MACHINE ??= "qemux86-64"
+MACHINE ?= "raspberrypi2"
```
RaspberryPi4 とかを使っている場合は、"raspberrypi4-64"としてください。

以上で、準備完了です！

## ビルドを実行する
ビルドには、上で出てきた"bitbake"コマンドを使います。
最小限イメージは以下のコマンドでビルドします。
```
$ bitbake core-image-minimal
```
ここでしばらく、数時間、５時間とか待つので、スリープしないようにして気長に待ちまます。
何をしているかというと、
```
$ bitbake core-image-minimal
Loading cache: 100% |                              | ETA:  --:--:--
Loaded 0 entries from dependency cache.
Parsing recipes: 100% |#####################################################################################| Time: 0:00:40
Parsing of 935 .bb files complete (0 cached, 935 parsed). 1838 targets, 74 skipped, 0 masked, 0 errors.
```
Parsing recipies ってところは、BSPレイヤー（meta-xxx）の中にあるレシピ（.bbファイル）を解析して、ビルド対象のOSSを決めて、ビルド順序を決めています。

そのまま放置すると、以下のような表示になります。
```
Currently  4 running tasks (429 of 3408)  12% |#########
0: gcc-cross-arm-13.1.1-r0 do_compile - 1m36s (pid 37037)
1: perl-native-5.36.1-r0 do_compile - 58s (pid 63069)
2: libffi-native-3.4.4-r0 do_configure - 30s (pid 71626)
3: re2c-native-3.0-r0 do_compile - 12s (pid 79170)
```
ビルド対象のOSSは3408個で、１つずつダウンロードして順番にビルドしているところです。
ここでは、RaspberryPiのビルドをするためのarmコンパイラを作っているところですね。
そりゃまぁ、時間はかかります。

PCがスリープしないように気をつけて、休憩ください。

## RaspberryPiに焼いてみる

