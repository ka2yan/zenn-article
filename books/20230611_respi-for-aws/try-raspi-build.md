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
YoctoがサポートしているLinux 環境を確認します。
[Yocto公式ページ](https://docs.yoctoproject.org/ref-manual/system-requirements.html?highlight=ubuntu#supported-linux-distributions)

今回は、Ubuntu22.04 を使用します。

Macのターミナルから、Docker でとってきます。
```
$ docker pull ubuntu:22.04
```
思ったより、すぐに完了します。

イメージが取得できていることを確認しましょう。
```
$ docker image ls
REPOSITORY              TAG       IMAGE ID       CREATED        SIZE
ubuntu                  22.04     3f5ef9003cef   1 days ago    69.2MB
```

コンテナ（実体）を作成します。
```
$ docker run -it -d --name raspi2-env ubuntu:22.04
```

作成できたか確認します。
```
$ docker container ps
CONTAINER ID   IMAGE          COMMAND       CREATED          STATUS          PORTS     NAMES
b7c4cb3eeb5c   ubuntu:22.04   "/bin/bash"   48 seconds ago   Up 47 seconds             raspi2-env
```

コンテナの中に入ります。
```
$ docker exec -it raspi2-env /bin/bash
root@b7c4cb3eeb5c:/#  
```

ここで実行するコマンドは、Ubuntuの中で実行するコマンドになります。
```
root@b7c4cb3eeb5c:/#  uname -a
Linux b7c4cb3eeb5c 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022 aarch64 aarch64 aarch64 GNU/Linux
root@b7c4cb3eeb5c:/# ls
bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@b7c4cb3eeb5c:/# 
```

コンテナでの作業の終了の仕方は以下です。
```
root@b7c4cb3eeb5c:/# exit
＄ 
```

コンテナの停止
```
$ docker stop raspi2-env
```

コンテナの開始
```
$ docker start raspi2-env
```
開始した後は、再度、exec コマンドで中に入りましょう。


## 必要なコマンドをUbuntu環境にインストールします
## Yoctoビルドシステムを導入する
## ビルドを実行してみる
## RaspberryPiに焼いてみる

