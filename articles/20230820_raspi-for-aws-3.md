---
title: "RaspberryPiをクラウドとちゃんと話せるようにする（第３回）"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["raspberrypi", "yocto", "aws", "docker"]
publication_name: "singularity"
published: true

---
組込みエンジニア katsu です。

RaspberryPiをクラウドとちゃんと話せるようにする、第２回です。
全体の位置付けは以下となります。

# [第１回](https://zenn.dev/singularity/articles/20230611_raspi-for-aws-1)
お試しで、RaspberryPiをAWS IoTと繋げてみます。
AWSに公開されている手順で、RaspberryPiとAWS IoTサービスを繋げて、メッセージの送受信をしてみます。

# [第２回](https://zenn.dev/singularity/articles/20230628_raspi-for-aws-2)
RaspberryPiをゼロからビルドできる環境を構築します。
DockerでUbuntu環境を立ち上げ、Yoctoというビルドシステムを用いてビルドします。
初めてビルドすると、一晩ぐらいかかります。大変。

# 第３回（今回ココ）
AWS IoTサービス向けに必要な機能を組込みます。
AWSが提供しているAWS IoT向けレシピ集 meta-aws をYoctoビルドシステムに組み込むことで、AWS接続できる機能を組み込むことができます。

# 第４回
RaspberryPiとAWS IoTサービスとメッセージの送受信をしてみます。
好きなメッセージをどうやって送るのか、AWS上でどうやって受信するのか、これを製品に仕立てるには何が必要なのか、確認します。

# 第５回
AWSに接続するために必要な情報をどうやって組み込むのか考えます。
AWSと接続するために必要な情報は何か、RaspberryPiに組み込むには、どのような方法があるのか、考察します。

----
# 第3回：  RaspberryPi に AWS IoT に必要な機能を追加する

RaspberryPiにAWS IoT接続に必要な機能を組み込んで、AWS IoTと接続するまでを確認します。
今回は、AWSと接続する際に必要な証明書や鍵情報を静的に組み込みますが、
次回以降で、動的に組み込むことにもチャレンジしてみたいと思います。

## meta-aws とは
meta-aws とは、AWSと通信するためのコマンドやライブラリを組み込むためのYoctoビルドシステム向けレシピ集（BSPレイヤー）です。本レシピ集使用することで、AWS IoTと接続するのに必要なライブラリやコマンドをRaspberryPiに組み込むことができます。

[meta-aws公式ページ](https://github.com/aws4embeddedlinux/meta-aws)

公式ページを参照すると、複数のAWS向けサービスを組み込むことができることが書いてあります。
本章では、AWS IoTと接続するための最小セットである
"AWS IoT Device SDK for C++ v2"とこれを使用するためのサンプルプログラムを組み込みます。

### aws-iot-device-sdk-cpp-v2 とは
”aws-iot-device-sdk-cpp-v2"とは、AWS IoT と接続するソフトウェア向けのC++ SDKです。
Python用SDKなどもありますが、ここではバイナリサイズが小さく、高速動作できる（であろう）C++ SDKを組み込みます。
SDK自体とサンプルプログラムのレシピ（bbファイル）は[こちら](https://github.com/aws4embeddedlinux/meta-aws/tree/master/recipes-sdk/aws-iot-device-sdk-cpp-v2)にあります。

## SDK の組込み
### SDKとサンプルコマンドをビルドする
[meta-aws公式ページ](https://github.com/aws4embeddedlinux/meta-aws) の Dependencies に記載に従って、OpenEmbeddedのBSPレイヤーと一緒に組み込んでいきます。

環境構築は、[前回の記事](https://zenn.dev/singularity/articles/20230628_raspi-for-aws-2)を参照ください。

meta-openembedded, meta-aws の２つのBSPレイヤーを git clone します。
```
$ cd raspi2/poky
$ git clone https://github.com/openembedded/meta-openembedded
$ git clone https://github.com/aws4embeddedlinux/meta-aws
```

”bblayer.conf"ファイルを開いて、
ビルド対象に、git cloneしたBSPレイヤーを追加します。
```diff bash:build/conf/bblayer.conf
@ -10,4 +10,9 @@
   /home/hoge/raspi2/poky/meta-poky \
   /home/hoge/raspi2/poky/meta-yocto-bsp \
   /home/hoge/raspi2/poky/meta-raspberrypi \
+  /home/hoge/raspi2/poky/meta-openembedded/meta-oe \
+  /home/hoge/raspi2/poky/meta-openembedded/meta-multimedia \
+  /home/hoge/raspi2/poky/meta-openembedded/meta-networking \
+  /home/hoge/raspi2/poky/meta-openembedded/meta-python \
+  /home/hoge/raspi2/poky/meta-aws \
```

続いて、”local.conf"ファイルを開いて、以下の名称でSDKとサンプルプログラムをビルド対象に加えます。

|  | 名称 |
| ---- | ---- |
| SDK | aws-iot-device-sdk-cpp-v2 |
| サンプルプログラム | aws-iot-device-sdk-cpp-v2-samples-mqtt5-pubsub |

実は、サンプルプログラムだけを追加しても、Yoctoで依存関係を見てくれて自動的にSDKも組み込んでくれるのですが、ここでは分かりやすくするために、両方書いておきます。

```diff bash:build/conf/local.conf
@@ -281,3 +281,15 @@
 # track the version of this file when it was generated. This can safely be ignored if
 # this doesn't mean anything to you.
 CONF_VERSION = "2"
+
+IMAGE_INSTALL:append = " aws-iot-device-sdk-cpp-v2"
+IMAGE_INSTALL:append = " aws-iot-device-sdk-cpp-v2-samples-mqtt5-pubsub"
"
```

ついでに、SSHも使えるようにしておきましょう。
```diff bash:build/conf/local.conf
-293,3 +293,6 @@
+IMAGE_INSTALL:append = " openssh openssh-sftp-server"
```

bitbakeコマンで、ビルドします。
```
＄ bitbake core-image-minimal
Loading cache: 100% |                                | ETA:  --:--:--
Loaded 0 entries from dependency cache.
Parsing recipes: 100% |##############################| Time: 0:01:43
Parsing of 2751 .bb files complete (0 cached, 2751 parsed). 4507 targets, 171 skipped, 0 masked, 0 errors.
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

Initialising tasks: 100% |#########################| Time: 0:00:01
Sstate summary: Wanted 427 Local 405 Mirrors 0 Missed 22 Current 1222 (94% match, 98% complete)
Removing 4 stale sstate objects for arch raspberrypi2: 100% |#####| Time: 0:00:00
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 3792 tasks of which 3742 didn't need to be rerun and all succeeded.
```
前回は3408個のタスクが、今回は3792個に増えました。

ビルドしたイメージのファイル名を確認します。
```
ls -l tmp/deploy/images/raspberrypi2/*.wic.bz2 
-rw-r--r-- 2 hoge hoge 26771677 Jul 17 08:26 tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2-20230716231341.rootfs.wic.bz2
lrwxrwxrwx 2 hoge hoge       61 Jul 17 08:26 tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2.wic.bz2 -> core-image-minimal-raspberrypi2-20230716231341.rootfs.wic.bz2
```
26,771,677バイト（26MB）のイメージファイルができました。

### RaspberryPiにイメージファイルを焼く
前回と同様、
Mac側のターミナルから、"docker cp" コマンドでイメージファイルをとってきます（解凍します）。
```
$docker ps
CONTAINER ID   IMAGE          COMMAND       CREATED       STATUS       PORTS     NAMES
b7c4cb3eeb5c   ubuntu:22.04   "/bin/bash"   3 weeks ago   Up 2 weeks             raspi2-env
$
$ docker cp b7c4cb3eeb5c:/home/hoge/raspi2/poky/build/tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2-20230716231341.rootfs.wic.bz2 .
$
$ bunzip2 core-image-minimal-raspberrypi2-20230716231341.rootfs.wic.bz2
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

### RaspberryPiを起動する
作成したイメージには ssh を組み込んだので、IPアドレス直指定（ここでは192.168.1.10とします）で接続しましょう（IPアドレスがわからない時は、ディスプレイ・キーボードをRaspberryPiに接続してキーボードから入力してください）。
Mac側のターミナルから、rootアカウントで接続してください。
```
＄ ssh root@192.168.1.10
```

mqtt5_pubsub コマンドが組み込まれているか、確認します。
```
# mqtt5_pubsub
Usage:
mqtt5-pubsub --ca_file <path> --cert <path> --client_id <str> --count <int> --endpoint <str> --help  --is_ci <str> --key <path> --log_file <str> --message <str> --port_override <int> --proxy_host <str> --proxy_port <int> --topic <str> --verbosity <log level>

....
```
Usageメッセージが表示されればOKです。
サンプルコマンドがちゃんと組み込まれていることが確認できました。

## AWS クラウド側を準備する
### AWSマネジメントコンソールからログイン
[AWSコンソール](https://aws.amazon.com/jp/console/)からログインします。
![](/images/20230611_110.png)
もし、AWSを初めて触る方は、以下あたりを参考に、AWSアカウントを作成してください。
[初めてのAWSユーザーですか？](https://docs.aws.amazon.com/ja_jp/accounts/latest/reference/welcome-first-time-user.html)

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
start.sh の説明は無視して、「サブスクリプション sdk/test/python」部分までスクロールします。
![](/images/20230819_140.png)
AWS側の設定は一旦これで終了です。

#### Step6. AWSのルートCA証明書をダウンロード
公式手順にはありませんが、ここでAWSが公開指定しているルート証明書をダウンロードします。
Mac上で新しいターミナルを開いて、以下のcurlコマンドで "ca-certificates.crt"という名前で保存します。
あとで説明しますが、必ずこのファイル名で保存しておいてください。
```
$ curl https://www.amazontrust.com/repository/AmazonRootCA1.pem > ca-certificates.crt
```

## RaspberryPiからメッセージを送信する。
Step4, Step6 でダウンロードしたZIPファイルとルートCA証明書を、RaspberryPiに持っていきます。
Mac上で新しいターミナルを開いて、/home/rootディレクトリに scp します。
```
$ scp connect_device_package.zip ca-certificates.crt root@192.168.1.10:/home/root
connect_device_package.zip             100% 6389   840.6KB/s   00:00    
ca-certificates.crt                    100% 1188   270.9KB/s   00:00 
```

&nbsp;
続いて、ssh で、RaspberryPiにログインします。
```
$ ssh root@192.168.1.10
```

scpしたZIPファイルを/etc/ssl/certsフォルダで解凍します。
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

MQTT5のサンプルコマンド実行時のヘルプを確認します。
```
mqtt5_pubsub  --endpoint <endpoint> --cert <path to the certificate> --key <path to the private key> --topic <topic name>
```

ここで重要なのは以下の６つの情報です。整理します。
|      | 情報 |
| ---- | ---- |
| ルートCA証明書   | ca-certificates.crt |
| デバイス証明書   | raspi2-test.cert.pem |
| プライベートキー | raspi2-test.private.key |
| エンドポイント | aqxxxx-ats.iot.ap-northeast-1.amazonaws.com  |
| クライアントID | basicPubSub |
| トピック | sdk/test/python |

エンドポイント、クライアントID、トピック は、start.sh の最終行に記載してあります。

MQTT5サンプルコマンドを参考にコマンド実行します。
後述しますが、ルートCA証明書はデフォルト名にしてありますので、指定不要です。
```
root@raspberrypi2:/etc/ssl/certs# mqtt5_pubsub --endpoint aqxxxx-ats.iot.ap-northeast-1.amazonaws.com --cert raspi2-test.cert.pem --key raspi2-test.private.key --client_id basicPubSub --topic sdk/test/python
Mqtt5 Client attempting connection...
Mqtt5 Client connection succeed, clientid: basicPubSub.
Subscription Success.
Publish Succeed.
Publish received on topic sdk/test/python:"Hello World 1"
Publish Succeed.
Publish received on topic sdk/test/python:"Hello World 2"
Publish Succeed.
Publish received on topic sdk/test/python:"Hello World 3"
Publish Succeed.
Publish received on topic sdk/test/python:"Hello World 4"
Publish Succeed.
Publish received on topic sdk/test/python:"Hello World 5"
Publish Succeed.
Publish received on topic sdk/test/python:"Hello World 6"
Publish Succeed.
Publish received on topic sdk/test/python:"Hello World 7"
Publish Succeed.
Publish received on topic sdk/test/python:"Hello World 8"
Publish Succeed.
Publish received on topic sdk/test/python:"Hello World 9"
Publish Succeed.
Publish received on topic sdk/test/python:"Hello World 10"
Mqtt5 Client disconnection with reason: libaws-c-mqtt: AWS_ERROR_MQTT5_USER_REQUESTED_STOP, Mqtt5 client connection interrupted by user request..
Mqtt5 Client stopped.
```
１秒毎に10回、メッセージを送りました。
最後、AWS_ERROR_MQTT5_USER_REQUESTED_STOP という表示が出ていますが、
サンプルコマンドは特に指定しなければ、10回コマンドを投げて終了するので、気にする必要はありません。

AWS側の設定ページを確認しましょう。
![](/images/20230819_150.png)
Hello worldが10回表示されてます！


次は、任意のメッセージを送ってみましょう。
ここでは、"I am RaspberryPi"というメッセージを1回だけ、送ってみます。 
--count, --message オプションをコマンドの最後に付与して、以下のように実行します。
```
root@raspberrypi2:/etc/ssl/certs# mqtt5_pubsub --endpoint aqxxxx-ats.iot.ap-northeast-1.amazonaws.com --cert raspi2-test.cert.pem --key raspi2-test.private.key --client_id basicPubSub --topic sdk/test/python --count 1 --message "I am RaspberryPi "
```

AWS側の設定ページを確認しましょう。
![](/images/20230819_160.png)
やりました。指定したメッセージをAWS IoT側に送る事ができました！

今回は、meta-awsのレシピを使って、RaspberryPi向けのC++SDKを組み込んで、
AWS IoT CoreへMQTTメッセージを送信できるところまでを確認しました。


以下は、今回のトラブルシューティングです。興味ある方はご覧ください。

## トラブルシューティング
mqtt5_pubsub コマンドですが、AWSクラウド側が準備できていなかったり、正しくコマンドを実行しないと、以下のようなエラーを出力します。
```
root@raspberrypi2:/home/root# mqtt5_pubsub --endpoint aqxxxx-ats.iot.ap-northeast-1.amazonaws.com --ca_file ca-certificates.crt --cert test0817-2.cert.pem --key test0817-2.private.key --client_id basicPubSub --topic sdk/test/python
Failed to Init Mqtt5Client with error code 1173: 
aws-c-io: AWS_IO_TLS_ERROR_DEFAULT_TRUST_STORE_NOT_FOUND, 
Default TLS trust store not found on this system. 
Trusted CA certificates must be installed, or 
"override default trust store" must be used while creating the TLS context.
```
今回、トラブルシューティングにどハマりしてしまったので、以下にまとめておきます。

### １） AWS_IO_TLS_ERROR_DEFAULT_TRUST_STORE_NOT_FOUND
#### エラーメッセージ
```
Failed to Init Mqtt5Client with error code 1173: 
aws-c-io: AWS_IO_TLS_ERROR_DEFAULT_TRUST_STORE_NOT_FOUND, 
Default TLS trust store not found on this system. 
Trusted CA certificates must be installed, or 
"override default trust store" must be used while creating the TLS context.
```
#### 原因
ルートCA証明書、ここではAWSのルート証明書を読み込めなかった。
#### 対策
証明書をデフォルトパスに置くか、引数で証明書を指定する必要があります。
mqtt5_pubsubコマンドは "--ca_file" でルートCA証明書を指定できますが、2023−08−19時点ではこのオプション指定を認識しませんでした。
以下を見ると、最新のca-certificates packageを使えば指定できるっぽいです（間違っているかもしれませんが）。
[TLS configuration with AWS CRT client](https://github.com/aws/aws-sdk-java-v2/discussions/3898#discussioncomment-5612167)
ただ、Yoctoで組み込んでいるca-certificates packageは、poky/meta/receipes-support/ca-certificates/ca-certificates_20211016.bb という風に2021年ものなので、これを最新にするのは面倒な（気がした）ので、デフォルトパスに証明書を置くようにします。
違っているようでしたら、ご指摘ください。

ですが、エラーログが示すデフォルトパスがどこかは記載されていないので [aws-c-io](https://github.com/awslabs/aws-c-io/)の中を AWS_IO_TLS_ERROR_DEFAULT_TRUST_STORE_NOT_FOUND を頼りに調べると [ここ](https://github.com/awslabs/aws-c-io/blob/c4b661f44497b18201b56ffd200cc478441f6434/source/s2n/s2n_tls_channel_handler.c#L136C1-L136C31)にありました。
複数の定義はありましたが、順番に見るようなので最初のdebianのパスに置くことにします。
```
/etc/ssl/certs/ca-certificates.crt
```
このため、本章では "--ca_file"での指定はせずに、AWSのルートCA証明書を以下に置くようにしました。

### 2） AWS_ERROR_FILE_INVALID_PATH
#### エラーメッセージ
```
Failed to setup mqtt5 client builder with error code 44: 
aws-c-common: AWS_ERROR_FILE_INVALID_PATH, Invalid file path.
```
#### 原因
指定したファイルパスが見当たらない。
ここでは、デバイス証明書とプライベートキーのファイルパスがわからなかった。
#### 対策
mqtt5_pubsubの "--cert", "--key"で指定するファイルを、コマンド実行時のフォルダに置くか、フルパスで指定するかしてください。
本章では、コマンド実行時のフォルダにフォルダに置いているため、ファイル名のみを指定しています。

### 3） AWS_IO_TLS_ERROR_NEGOTIATION_FAILURE
#### エラーメッセージ
```
Mqtt5 Client connection failed with error: 
aws-c-io: AWS_IO_TLS_ERROR_NEGOTIATION_FAILURE, TLS (SSL) negotiation failed.
```
#### 原因
一番ハマったのがこの問題でした。
暗号化通信（TLS）接続時のネゴシエーションエラーになります。
このメッセージだけでは原因はわからないので、opensslコマンドを使って調べる事ができます。
ネットワークの問題なのか、証明書の問題なのか、色々と原因があるのですが、今回は、RaspberryPiの現在時刻が合っていない（昔の時刻になっている）というのが原因でした。
#### 対策
RaspberryPiのシステム時刻を合わせる必要があります。
UTC時刻で設定します。PCで表示されている時刻から9時間引いた時刻を以下のように設定します。
```
# date -s "2023-08-16 14:31:00"
```
このコマンドはLinux上の時刻を更新するだけで、ハードウェア側、RTC(Real Time Clock)には保存されません。
一般的には、RTCはボタン電池などで駆動し、常時時刻情報を更新しています。Linuxは起動時にRTCから現在時刻を取得し、シャットダウンや再起動時にRTCに情報を保存します。これにより、次回起動した時に、そこそこ正しい時刻で起動できる、という流れになっています。
手動でLinuxの時刻とRTCを同期するには、hwclockコマンドなどを使用しますが、残念ながらRaspberryPiはRTCを搭載してないので、このコマンドはエラーになります。
恒久対策としては、RTCを搭載するとか、NTPも組み込んで起動時にNTPサーバーと時刻を同期する、などがあります。

opensslコマンドでの調査ですが、こちらも一手間かかったので、いかに記載しておきます。

#### OpenSSLコマンドによるTLS接続エラー調査
aws-c-ioによるエラーですが、接続処理自体はRaspberryPiに組み込んでいるOpenSSLのライブラリが実行しています。このため、opensslコマンドで調査できます。
[AWSデベロッパーガイド](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/diagnosing-connectivity-issues.html)

ただ、、、
```
root@raspberrypi2:~# openssl
-sh: openssl: not found
```
今回ビルドしたイメージファイルには、opensslコマンド本体が組み込まれていません。
なので、opensslコマンドを組み込むという流れになります。
例えば、bitbakeコマンドで core-image-minimal 以外でビルドする、自分でレシピを追加するというのが良いと思いますが、今回は、面倒なので最低限のバイナリだけRaspberryPiにコピーしました。
（デバッグ用途なんで許してください・・・）

opensslだけ再ビルドします。
```
$ bitbake -c cleansstate openssl
$ bitbake core-image-minimal
```

以下がopenssl本体です。fileコマンドで確認します。
```
＄ file tmp/work/cortexa7t2hf-neon-vfpv4-poky-linux-gnueabi/openssl/3.1.1-r0/package/usr/bin/openssl 
tmp/work/cortexa7t2hf-neon-vfpv4-poky-linux-gnueabi/openssl/3.1.1-r0/package/usr/bin/openssl: ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, BuildID[sha1]=b43cf377ca6fad3d6bd5455382ca638ed66eb19d, for GNU/Linux 5.15.0, stripped
```

続いて、openssl, libssl.so.3 の2ファイルを、docker cpとscpコマンドでRaspberryPiに持っていきます。
デバッグ用途なので、/home/rootに保存します。
```
$ docker cp b7c4cb3eeb5c:/home/hoge/raspi2/poky/build/tmp/work/cortexa7t2hf-neon-vfpv4-poky-linux-gnueabi/openssl/3.1.1-r0/package/usr/bin/openssl .
$ docker cp b7c4cb3eeb5c:/home/hoge/raspi2/poky/build/tmp/work/cortexa7t2hf-neon-vfpv4-poky-linux-gnueabi/openssl/3.1.1-r0/package/usr/lib/libssl.so.3 .
$
$ scp openssl libssl.so.3 root@192.168.1.10:/home/root
```

opensslコマンドを実行してみます。
普通に実行するとlibssl.soを読み込めないというエラーが出るので、ライブラリのパスをLD_LIBRARY_PATH変数で指定します。
```
root@raspberrypi2:~# chmod +x /home/root/openssl
root@raspberrypi2:~# LD_LIBRARY_PATH=/home/root /home/root/openssl version
OpenSSL 3.1.1 30 May 2023 (Library: OpenSSL 3.1.1 30 May 2023)
```
ということで、無理矢理ですが、opensslコマンド無事実行できるようになりました。

では、openssl s_clientコマンドで AWSとの接続を確認します。
```
root@raspberrypi2:~# cd /etc/ssl/certs
root@raspberrypi2:/etc/ssl/certs# LD_LIBRARY_PATH=/home/root /home/root/openssl s_client -connect aqxxxx-ats.iot.ap-northeast-1.amazonaws.com:8443 -CAfile ca-certificates.crt -cert raspi2-test.cert.pem -key raspi2-test.private.key 
CONNECTED(00000003)
depth=2 C = US, O = Amazon, CN = Amazon Root CA 1
verify return:1
.
.
.
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256
Server public key is 2048 bit
This TLS version forbids renegotiation.
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 9 (certificate is not yet valid)
```
Verify return code: 0 以外の場合、エラーとなります。
Verify return code: 9 を検索すると、システム時刻が古いくて証明書がまだ有効でない、という意味になる、とのことです。

現在時刻を設定します。
```
root@raspberrypi2:/etc/ssl/certs# date -s "2023-08-20 6:09"
```

再度、opensslコマンドを実行します。
```
root@raspberrypi2:/etc/ssl/certs# LD_LIBRARY_PATH=/home/root /home/root/openssl s_client -connect aqxxxx-ats.iot.ap-northeast-1.amazonaws.com:8443 -CAfile ca-certificates.crt -cert raspi2-test.cert.pem -key raspi2-test.private.key 
CONNECTED(00000003)
depth=2 C = US, O = Amazon, CN = Amazon Root CA 1
verify return:1
.
.
.
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256
Server public key is 2048 bit
This TLS version forbids renegotiation.
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
```
最終行のreturn code: 0(ok)となり、無事AWS側と接続できたことを確認できました。

## まとめ
今回は、meta-awsのレシピを使って、RaspberryPi向けのC++SDKを組み込んで、
AWS IoT CoreへMQTTメッセージを送信できるところまでを確認しました。
次回は、AWS　IoT　Coreをきちんと設定して、MQTTメッセージの送受信とアクション実行を確認していきたいと思います。

## 参照
AWS IoT におけるデバッグ方法
https://pages.awscloud.com/rs/112-TZM-766/images/EV_aws-iot-deep-dive-3-310-topic1_Mar-2021.pdf

