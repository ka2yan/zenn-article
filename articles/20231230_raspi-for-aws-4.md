---
title: "RaspberryPiをクラウドとちゃんと話せるようにする（第４回）"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["raspberrypi", "yocto", "aws", "docker"]
publication_name: "singularity"
published: true

---
組込みエンジニア katsu です。

RaspberryPiをクラウドとちゃんと話せるようにする、第４回です。
全体の位置付けは以下となります。

# [第１回](https://zenn.dev/singularity/articles/20230611_raspi-for-aws-1)
お試しで、RaspberryPiをAWS IoTと繋げてみます。
AWSに公開されている手順で、RaspberryPiとAWS IoTサービスを繋げて、メッセージの送受信をしてみます。

# [第２回](https://zenn.dev/singularity/articles/20230628_raspi-for-aws-2)
RaspberryPiをゼロからビルドできる環境を構築します。
DockerでUbuntu環境を立ち上げ、Yoctoというビルドシステムを用いてビルドします。
初めてビルドすると、一晩ぐらいかかります。大変。

# [第3回](https://zenn.dev/singularity/articles/20230628_raspi-for-aws-3)
AWS IoTサービス向けに必要な機能を組込みます。
AWSが提供しているAWS IoT向けレシピ集 meta-aws をYoctoビルドシステムに組み込むことで、AWS接続できる機能を組み込むことができます。

# 第4回（今回ココ）
RaspberryPiとAWS IoTサービスとメッセージの送受信をしてみます。
RaspberryPi視点で、好きなメッセージをどうやって送るのか、AWS上でどうやって受信するのか、確認します。

# 第5回
RaspberryPiとAWS IoTサービスとメッセージの送受信をしてみます。
AWS視点で証明書管理をどうするのか、受信したメッセージをどうアクションに繋げるにはどうするのか、を確認していきたいと思います。

# 第6回
AWSに接続するために必要な情報をどうやって組み込むのか考えます。
AWSと接続するために必要な情報は何か、RaspberryPiに組み込むには、どのような方法があるのか、考察します。

----
# 第４回：  AWS IoT Coreとメッセージ送受信するプログラムを作る

前回は、RaspberryPiにAWS IoT接続に必要な機能を組み込んで、接続確認までを行いました。
この時はAWSとのメッセージの送受信は、mqtt5_pubsubというAWSが公開しているサンプルコマンドを使用しました。
ただ、このサンプルコマンドは、１秒毎に番号付きのメッセージを送信して、送信したメッセージをAWS経由受信するプログラムでした。
今回は、このサンプルコマンドを変更して、RaspberryPi側で以下を実現します。
- 好きな宛先（任意のtopic）に、好きな時に好きなMQTTメッセージを送る
- 自分宛（自分のtopic）に届いた MQTTメッセージを表示する

## MQTTとは

![](/images/20231225_005.png)

MQTT（Message Queueing Telemetry Transport）とは、TCP/IPをベースとしたIoT向けのプロトコルです。
機械同士が通信を介して情報をやり取りするM2M(Machine-to-Machine)や、家電や自動車など多種多様な「モノ」がインターネットにつながり、お互いに情報をやり取りするIoT(Internet of Things)で使われることを想定しています。

MQTTは一方向、1対1の通信のみでなく、双方向、1対多の通信が可能でありながら、プロトコルヘッダ２バイトからととてもが小さいのが特徴で、通信量やCPU負荷、電力消費量などをHTTPなど比べて低減することができます。

また、証明書などを使用した認証プロトコルを使用して、メッセージを暗号化してサーバーと通信することでセキュアな運用をすることができます。

### Publish／Subscribeモデル
また、Publish/Subscribeモデルを採用しています。このモデルでは、メッセージの送信側をPublisher、メッセージの受信側をSubscriberに分け、MQTT Brokerがメッセージの中継を行います。
PublisherはSubscriberを意識することなく、Brokerにメッセージの送信ができ、Brokerはそれらのメッセージを預り、適切にSubscriberに配信する責任を持ちます。
Publish/Subscribeする宛先は、フォルダなどと同じように階層化されて表現され、topicと言います。
家の１階にあるセンサー１を表すtopicは、例えば、"home/floor1/sensor1"などと表すことができます。

### MQTTサンプルプログラム（mqtt5_pubsub）
今回は meta-aws にある aws-iot-device-sdk-cpp-v2 を使用するMQTTサンプルプログラム [mqtt5_pubsub（GitHub）]((https://github.com/aws/aws-iot-device-sdk-cpp-v2/blob/main/samples/mqtt5/mqtt5_pubsub/main.cpp)) を使用します。
このサンプルプログラムの特徴は以下です。
- 指定した証明書と秘密鍵を使ってAWS IoT Coreに接続する
- 指定したtopicに、1秒に1回、決まったメッセージを決まった回数 送信（publish）する
- 指定したtopicから受信（Subscribe）したメッセージを表示する（自分が送ったメッセージも受信します）
- publish & subscribe topicは同じものを使用する

今回は、このサンプルプログラムを好きな時に好きなtopicとメッセージを送れるようにします。

## システム構成
システム構成は、送信デバイスがRaspberryPi、MQTT BrokerがAWS IoT Coreとなります。
RaspberryPiには、mqtt5_pubsubコマンドが組み込まれており、topic_AにメッセージをPublish（送信）し、topic_BでメッセージをSubscribe（受信）をできるようにします。

![](/images/20231225_010.png)


## ブロック図
RaspberryPi側の機能を、AWSと通信するのはmqtt5_pubsubコマンドと、メッセージを送受信をする自前プログラム（awsappコマンド）に分けます。
awsappコマンドは、mqtt5_pubsubとUNIXドメインソケット（socket_A, socket_B）を使用してメッセージを送受信します。
Socket_AにPublishするメッセージを送信し、Socket_BからSubscribeしたメッセージを受信します。

![](/images/20231225_011.png)

ここでは、socket_A は”/tmp/aws_socket_smsg"、socket_Bは”/tmp/aws_socket_rmsg"という名前にします。


## mqtt5_pubsubの修正
mqtt5_pubsubコマンドに対して以下の２点の修正を入れます。
- 起動時に指定されたtopicでメッセージを待ち受けし、Subscribe(受信)したらソケットAに送る。
- ソケットBからtopic と メッセージを受け取り、AWS側にPublish（送信）する。

修正パッチは[0001-mqtt5_pubsub.patch（Github）](https://github.com/ka2yan/meta-awsapp/blob/main/recipes-sdk/aws-iot-device-sdk-cpp-v2/files/0001-mqtt5_pubsub.patch)におきました。
大きく３つの修正を入れてあり、
１つ目がSubscribeしたメッセージをソケットに出力する部分
２つ目がPublishするメッセージを受信するためのソケットを待ち受ける部分
３つ目がPublishするメッセージを受信し、指定されたtopicにメッセージを送信する部分です。
Subscribeしたメッセージを出力するソケットは、繋げっぱなしで受信する度にソケットに出力します。
また、Publishメッセージは1回ずつソケットをクローズするようにしました。

## awsappの実装
自前のawsappコマンドは、ソケットからメッセージを送受信するプログラムになります。
機能は、以下になります。
- 起動時に指定されたソケットAで待ち受けし、受信したメッセージを出力する。
- 指定したソケットBに対して、ファイルに格納されているtopicとメッセージを送信する。

コードは[awsapp（Github）](https://github.com/ka2yan/meta-awsapp/blob/main/recipes-awsapp/awsapp/files/main.c)におきました。

ここでは、コマンド仕様を記載します。
```sh:受信したメッセージを10,000回表示するとき
awsapp recv /tmp/aws_socket_rmsg  10000
```

```sh:topicとメッセージを格納したファイルを送信するとき
awsapp send /tmp/aws_socket_smsg ./message.txt
```

メッセージファイルの仕様は、１行目がtopic、２行目以降がメッセージとします。
topicが"house/floor1/sensor1"の JSONメッセージの例を以下に記載します。
```
house/floor1/sensor1
{
  "message": "Hello from Raspberry Pi"
}
```

## Yocto組み込み
続いて、これらのプログラムをRaspberryPiのYoctoシステムに組み込みます。
環境構築は、[前回](https://zenn.dev/singularity/articles/20230820_raspi-for-aws-3)を参照ください。

### meta-awsapp
上記のmqtt5_pubsubコマンドのパッチと、awsappコマンドを含んだBSPレイヤーを以下に作成しました。

https://github.com/ka2yan/meta-awsapp

#### meta-awsapp レイヤーの詳細
meta-awsapp　レイヤーは、以下のように大きく２つに分けました。

| コマンド | パス |
| ---- | ---- |
| awsappコマンド  | recipes-awsapp/awsapp |
| mqtt5_pubsubパッチ | recipes-sdk/aws-iot-device-sdk-cpp-v2 |

```
meta-awsapp
|-- COPYING.MIT
|-- README
|-- conf
|   `-- layer.conf
|-- recipes-awsapp
|   `-- awsapp
|       |-- awsapp_1.0.bb
|       `-- files
|           |-- Makefile
|           `-- main.c
`-- recipes-sdk
    `-- aws-iot-device-sdk-cpp-v2
        |-- aws-iot-device-sdk-cpp-v2-samples-mqtt5-pubsub.bbappend
        `-- files
            `-- 0001-mqtt5_pubsub.patch
```

###### recipes-awsapp/awsapp
awsappをビルドするレシピ（awsapp_1.0.bb）とソースコードを置くディレクトリ（files）になります。
レシピは以下となります。

```sh:awsapp_1.0.bb
SUMMARY = "AWS Application"
DESCRIPTION = "C Application working with AWS IoT Core sample app(mqtt5_pubsub)."
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://."

S = "${WORKDIR}"

TARGET_CC_ARCH += "${LDFLAGS}"

do_install() {
	install -d ${D}${bindir}
	install -m 0755 awsapp ${D}${bindir}
}
```

| 変数 | 内容 |
| ---- | ---- |
| SUMMARY | 任意の文字列でOKです。”My Application"とします。 |
| DESCRIPTION | 任意の文字列でOKです。”My Application in C"とします。 |
| LICENSE | サンプルと同じMIT Licenseとします<br>他のライセンスでも、自前のライセンスでも良いと思います。 |
| LIC_FILES_CHKSUM | exampleのレシピにはありませんでしたが、組み込むライセンスファイルのmd5sum値を指定します。<br>ひとまずこのままお使いください。<br>もしビルド時にエラーとなっても、正解を教えてもらえます。 |
| SRC_URI | ソースコードのパスを指定します。<br>レシピと同一フォルダなので"file://."となります。<br>外部のgithubのリポジトリであれば、ここにgithub上のURLを記載します。 |
| S | ソースコードがあるフォルダを指定します。<br>WORKDIRとしておくと、使いそうなフォルダ名を調べてくれます。<br>filesフォルダを自動で見つけてくれます。 |
| TARGET_CC_ARCH | コンパイル時に、Yoctoビルドシステムで使うLDFLAGSを渡すようにします。<br>これを指定しないと、ビルド時にエラーとなります。 |
| do_install | 作成したプログラムをファイルシステム上のどこに置くかを指定します。<br>ここでは/bin下にmyappを置くようにします。 |

コンパイル時のCC変数やコンパイルオプションなどは、なんとYoctoビルドシステムのものが勝手に継承されてます。
do_compile() などで指定もできますが、特別なビルドが必要なければ、定義すら不要です。
こんな手抜きで、クロスコンパイルができるなんで、なんて幸せな世の中でしょう。

files下のMakefileは、all, cleanぐらいを準備しておけばOKです。
```makefile:Makefile
TARGET=awsapp
SRCS=main.c

all:
	${CC} ${SRCS} -o ${TARGET}

clean:
	rm -f {TARGET}
```

###### recipes-sdk/aws-iot-device-sdk-cpp-v2
mqtt5_pubsubにパッチを当てるレシピ（aws-iot-device-sdk-cpp-v2-samples-mqtt5-pubsub.bbappend）とパッチを置くディレクトリ（files）になります。

```
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI:append = " file://0001-mqtt5_pubsub.patch"
```

| 変数 | 内容 |
| ---- | ---- |
| FILEEXTRAPATHS:prepend | 追加レシピで使用するファイル置き場のパス |
| SRC_URI:append | 差分コード記載されたパッチファイル |

files下におくパッチは、修正したコードを git diff したものでOKです。

&nbsp;

#### Yoctoビルドシステムに組み込む
では、meta-awsappを実際に組み込んでいきましょう。

Yoctoのトップディレクトリに移動して、git clone します。
```
$ cd raspi2/poky
$ git clone https://github.com/ka2yan/meta-awsapp
```

”bblayer.conf"ファイルを開いて、
ビルド対象に、git cloneしたBSPレイヤーを追加します。
meta-aws の下に追加します。
```diff bash:build/conf/bblayer.conf
@ -10,4 +10,9 @@
   /home/hoge/raspi2/poky/meta-poky \
   /home/hoge/raspi2/poky/meta-yocto-bsp \
   /home/hoge/raspi2/poky/meta-raspberrypi \
   /home/hoge/raspi2/poky/meta-openembedded/meta-oe \
   /home/hoge/raspi2/poky/meta-openembedded/meta-multimedia \
   /home/hoge/raspi2/poky/meta-openembedded/meta-networking \
   /home/hoge/raspi2/poky/meta-openembedded/meta-python \
   /home/hoge/raspi2/poky/meta-aws \
+  /home/hoge/raspi2/poky/meta-awsapp \
```

”local.conf"ファイルを開いて、
ビルド対象に、awsappコマンドを追加します。
```diff bash:build/conf/local.conf
@@ -284,6 +284,7 @@
  IMAGE_INSTALL:append = " aws-iot-device-sdk-cpp-v2"
 IMAGE_INSTALL:append = " aws-iot-device-sdk-cpp-v2-samples-mqtt5-pubsub"
+IMAGE_INSTALL:append = " awsapp"
```

### ビルド
bitbakeコマンで、ビルドします。
```
$ cd build
$ source oe-init-build-env
....

$ bitbake core-image-minimal
Loading cache: 100% |##########################################Time: 0:00:00
Loaded 4508 entries from dependency cache.
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
meta-oe              
meta-multimedia      
meta-networking      
meta-python          = "master:bf314d2c57d63afd680ab21baf43e150456120ed"
meta-aws             = "master:c370b087fa54a2b2a01c8ecff6d4d05ea9d141bf"
meta-awsapp          = "master:13b646c0e167ca52f69c91be5538049b172015ac"

Initialising tasks: 100% |####################################| Time: 0:00:01
Sstate summary: Wanted 9 Local 0 Mirrors 0 Missed 9 Current 1648 (0% match, 99% complete)
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 3808 tasks of which 3792 didn't need to be rerun and all succeeded.
```
無事ビルドができました。

ビルドしたイメージのファイル名を確認します。
```
$ ls -l tmp/deploy/images/raspberrypi2/*.wic.bz2
-rw-r--r-- 2 hoge hoge 28981833 Dec 16 20:25 tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2-20231216112502.rootfs.wic.bz2
```
28,981,833バイト（28MB）のイメージファイルができました。

### RaspberryPiでの確認
前回と同様、
Mac側のターミナルから、"docker cp" コマンドでイメージファイルをとってきます（解凍します）。
```
$docker ps
CONTAINER ID   IMAGE          COMMAND       CREATED       STATUS       PORTS     NAMES
b7c4cb3eeb5c   ubuntu:22.04   "/bin/bash"   3 weeks ago   Up 2 weeks             raspi2-env
$
$ docker cp b7c4cb3eeb5c:/home/hoge/raspi2/poky/build/tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2-20231216112502.rootfs.wic.bz2 .
$
$ bunzip2 core-image-minimal-raspberrypi2-20231216112502.rootfs.wic.bz2
```

書き込みにも前章で使用した RaspberryPi Imager を使用しましょう。
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


## 動作確認
AWS側とRaspberryPi側をそれぞれ準備していきましょう。

### AWS の準備
### AWSマネジメントコンソールからログイン
[AWSコンソール](https://aws.amazon.com/jp/console/)からログインします。
![](/images/20230611_110.png)

### IoT Core に入る
ログインしたら上部にある検索欄に、”IoT"を入力して、”IoT Core"を選択します。
![](/images/20230611_120.png)
### デバイスを作成する
AWS IoT ページが表示されました。
左メニューにある「１個のデバイスを接続」を選択してください。
![](/images/20230611_130.png)
Step1,2,3,4 の手順に従って、AWSとRaspberryPiを設定していきましょう。
![](/images/20230611_150.png)
#### Step1. デバイスを準備する
上から順に読んでいくと、、、
![](/images/20230611_160.png)
「ターミナルウィンドウから、次のコマンドを入力します」というところで、
pingコマンドが記載してあります。
これは、RaspberryPiから、AWSのサーバーに接続できるのか、を確認してね、という意味です。
&nbsp;
では、コマンドをコピーして、RaspberryPiのターミナルから、実行します。
```
pi@raspberrypi2:~ $ ping aqxxxx.ap-northeast-1.amazonaws.com
PING aqxxxx.ap-northeast-1.amazonaws.com(2400:.... (2400:...)) 56 data bytes
64 bytes from 2400:... (2400:...): icmp_seq=1 ttl=250 time=5.30 ms
64 bytes from 2400:... (2400:...): icmp_seq=1 ttl=250 time=5.13 ms
^C
```
成功しました。
RaspberryPiからAWSまでの通信ができることが確認できました。
#### Step2. デバイスを登録して保護する
IoTデバイスとして、新しいモノ(Thing)を作成します。
名前は、"raspi2-test"とします。
![](/images/20230819_110.png)
#### Step3. プラットフォームとSDKを選択します
RaspberryPiのOSと、組み込むSDKの種類を選択します。
ここでは、Linux/macOS, Pythonを選びます。
![](/images/20230611_180.png)
#### Step4. 接続キットをダウンロード
接続キットをダウンロードします。
![](/images/20230819_130.png)

#### Step5. 接続キットを実行
デバイスから受信したメッセージを表示する画面になります。
接続キットは手に入ったので、このページでの動作確認はスキップします。
「続行」をクリックします。
![](/images/20230819_140.png)

#### Step6. AWSのルートCA証明書をダウンロード
公式手順にはありませんが、ここでAWSが公開指定しているルート証明書をダウンロードします。
Mac上で新しいターミナルを開いて、以下のcurlコマンドで "ca-certificates.crt"という名前で保存します。
あとで説明しますが、必ずこのファイル名で保存しておいてください。
```
$ curl https://www.amazontrust.com/repository/AmazonRootCA1.pem > ca-certificates.crt
```

&nbsp;

### RaspberryPi準備
Step4, Step6 でダウンロードしたZIPファイルとルートCA証明書を、RaspberryPiに持っていきます。
Mac上で新しいターミナルを開いて、/home/rootディレクトリに scp します。
```
$ scp connect_device_package.zip ca-certificates.crt root@192.168.1.10:/home/root
connect_device_package.zip             100% 6389   840.6KB/s   00:00    
ca-certificates.crt                    100% 1188   270.9KB/s   00:00 
```

&nbsp;
続いて、Macからターミナルを開いて、ssh で、RaspberryPiにログインします（ターミナル１）。
```
$ ssh root@192.168.1.10
```

scpしたZIPファイルを/etc/ssl/certsディレクトリで解凍します。
```
root@raspberrypi2:~# mkdir -p /etc/ssl/certs
root@raspberrypi2:~# cp connect_device_package.zip ca-certificates.crt /etc/ssl/certs
root@raspberrypi2:~# cd /etc/ssl/certs
root@raspberrypi2:/etc/ssl/certs# unzip connect_device_package.zip 
Archive:  connect_device_package.zip
  inflating: raspi2-test.cert.pem
  inflating: raspi2-test.public.key
  inflating: raspi2-test.private.key
  inflating: raspi2-test-Policy
  inflating: start.sh
```

AWSとの接続する際には、RaspberryPiの時刻が（大体でも）合っている必要があるので、
時刻を設定しておきます。
```
# date -s "2023-12-30 14:52:00"
```

AWSと接続する mqtt5_pubsub コマンドを実行して、メッセージを送受信待ちします。
```
root@raspberrypi2:/etc/ssl/certs# mqtt5_pubsub --endpoint aqxxxx-ats.iot.ap-northeast-1.amazonaws.com --cert raspi2-test.cert.pem --key raspi2-test.private.key --client_id basicPubSub --topic sdk/test/python
Mqtt5 Client attempting connection...
Mqtt5 Client connection succeed, clientid: basicPubSub.
Subscription Success.
```
テスト用の証明書を使用しているので、受信（Subscribe）するtopicは、”sdk/test/python"とします。
これで準備が終わりました。

&nbsp;

### メッセージの受信
では、AWS IoT Coreからメッセージを送信し、RaspberryPiでメッセージを受信します。
まず、Macから別のターミナルを開いて、受信したメッセージを表示するawsapp recvコマンドを実行します（ターミナル２）。
メッセージ受信回数はひとまず、10,000回としておきます。
```ssh:ターミナル２
$ ssh root@192.168.1.10
...
root@raspberrypi2:~# awsapp recv /tmp/aws_socket_rmsg 10000
wait message...
```
これで、mqtt5_pubsubコマンドが受信したメッセージがターミナル２に表示されます。

&nbsp;
続いて、AWSのIoT Coreページに戻って、「MQTTテストクライアント」を開きます。
![](/images/20231225_110.png)

「トピックに公開する」タブを選択し、トピック名”sdk/test/python"を指定して、「発行」します。
![](/images/20231225_120.png)

ターミナル２を見てください。
mqtt5_pubsubコマンドが受信したメッセージがソケットを通して、awsappコマンド側で受信できました。
```ssh:ターミナル２
Receive 45 Byte
{
  "message": "Hello from AWS IoT console"
}
wait message...
```

AWS側のページで、メッセージペイロードを変更して、発行してみてみましょう。
受信も発行して即時に受信していることがわかると思います。

これで好きなタイミングで、好きなメッセージを受信した処理を行うことができるようになりました。


### メッセージの送信
続いて、RaspberryPiからメッセージを送信し、AWS IoT Coreで受信します。
AWS側で、「トピックをサブスクライブする」タブを選択し、トピックのフィルター”sdk/test/java"を指定して、「サブスクライブ」します。
![](/images/20231225_140.png)

サブスクライブの下を見ると、”sdk/test/java"で待ち受けしているのがわかります。
![](/images/20231225_150.png)


続いて、更にもう一つのターミナルを開きます（ターミナル３）。
```
$ ssh root@192.168.1.10
```

送信するtopicとメッセージを含んだファイルを作りましょう。
AWS側でサブスクライブしているtopic "sdk/test/java"を使用します。
```
root@raspberrypi2:~# cat message.txt
sdk/test/java
{
  "message": "Hello from Raspberry Pi"
}
```
1行目がtopicで、2行目以下がメッセージになります。

awsapp sendコマンドで、AWSに対して作成したメッセージを送ります。
```
root@raspberrypi2:~# ./awsapp send /tmp/aws_socket_smsg ./message.txt 
send message to /tmp/aws_socket_smsg.
```

AWS側のページを見てください。
message.txtで記入したメッセージを受信できました。
![](/images/20231225_160.png)

メッセージファイルを色々変更して、topicを変えたりメッセージを変えてみてください。
これで、AWSの好きな宛先（topic）に、好きなメッセージを送信することができました。

## まとめ
今回は、AWSが公開しているMQTTプログラムmqtt5_pubsubコマンドを変更して、RaspberryPi側で以下を実現しました。
- 好きな宛先（任意のtopic）に、好きな時に好きなMQTTメッセージを送る
- 自分宛（自分のtopic）に届いた MQTTメッセージを表示する

次回は、AWS側で証明書やtopicの管理をどうするのか、受信したメッセージをどうアクションに繋げるにはどうするのか、を確認していきたいと思います。

