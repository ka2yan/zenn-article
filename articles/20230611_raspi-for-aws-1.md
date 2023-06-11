---
title: "RaspberryPiをクラウドとちゃんと話せるようにする（第１回）"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["raspberrypi", "yocto", "aws"]
publication_name: "singularity"
published: true

---
組込みエンジニア katsu です。

組込み機器をクラウドを繋げるのは簡単だけど、ちゃんとやるのは意外と難しいですよね。

例えば、RaspberryPiをAWS に繋げて、お試し的にメッセージをやり取りできるようにすることは簡単です。
でも、これを製品としてAWSに繋げようとすると、どうでしょう？
AWSと接続するために必要な機能をビルドして組み込んでおく必要があります。
AWSと接続するために必要な情報をなんとかして組み込む必要があります。
このように、製品にしようとした途端に難しくなってしまいます。

ここでは、以下５回に分けて、まずはRaspberrypPiを製品と見立てて、AWSと接続するために必要な機能をビルドして組み込むにはどうするか、AWSと接続するために必要な情報をどうやって組み込むか、手を動かして考えてみたいと思います。

# 第１回
お試しで、RaspberryPiをAWS IoTと繋げてみます。
AWSに公開されている手順で、RaspberryPiとAWS IoTサービスを繋げて、メッセージの送受信をしてみます。

# 第２回
RaspberryPiをゼロからビルドできる環境を構築します。
DockerでUbuntu環境を立ち上げ、Yoctoというビルドシステムを用いてビルドします。
初めてビルドすると、一晩ぐらいかかります。大変。

# 第３回
AWS IoTサービス向けに必要な機能を組込みます。
AWSが提供しているAWS IoT向けレシピ集 meta-aws をYoctoビルドシステムに組み込むことで、AWS接続できる機能を組み込むことができます。

# 第４回
RaspberryPiとAWS IoTサービスとメッセージの送受信をしてみます。
好きなメッセージをどうやって送るのか、AWS上でどうやって受信するのか、これを製品に仕立てるには何が必要なのか、確認します。

# 第５回
AWSに接続するために必要な情報をどうやって組み込むのか考えます。
AWSと接続するために必要な情報は何か、RaspberryPiに組み込むには、どのような方法があるのか、考察します。

----
# 第１回：  お試しで RaspberryPiをAWS IoTと繋げてみよう
![](/images/20230611_000.jpg)
早速やってみましょう。
第１回は、お試しとしてRaspberryPiをAWS IoT Core　サービスに繋げてみます。
具体的には、RaspberryPi を最新の状態にした後、
AWS IoT Coreと一緒に設定して、
RaspberryPiとお話ができるようにしていきます。

## 使用する機材
- Raspberry Pi 2
- micro-SD
- Macbook Air
  Raspberry Pi Imager, Terminal

## RaspberryPi を最新の状態にする
まずは、RaspberryPiで使用するSDカードに、最新のイメージを書き込みます。
Macに、SDカードを認識させましょう。
### RaspberryPi Imagerをインストールする
[RaspberryPi Imager公式ページ](https://www.raspberrypi.com/software/) にアクセスします。
macOS / Windows / Ubuntu 向けインストーラがあるので、ダウンロードしてインストールします。
### RaspberryPi Imagerで、SDカードにイメージを書き込む
RaspberryPi　Imagerを起動します。
![](/images/20230611_010.png)
書き込むOSを選択します。
「Raspberry Pi OS (32bit)」 を選択します。
ソフトは自分で追加できるので、特別な目的がない限り、このOSで良いと思います。
ストレージは、Macに接続したSDカードを選択します。
![](/images/20230611_020.png)
まだ、書き込みませんよw
&nbsp;
設定（歯車）ボタンを押してください。
以下の設定をしてください。
  - ホスト名（設定は必須）
  ここでは、raspberrypi2 とします。
  .localはローカルドメインというだけなので、今回はあまり気にしなくて良いです。
- SSHを有効化する（設定は必須）
  パスワード認証を使う、を選択し、ユーザー名をパスワードを指定します。ここでは、ユーザー名は "pi" にします。
  ユーザー名:pi
  パスワード:<password> 任意のパスワード
- WiFiを有効にする（WiFiモデルは必須）
　RaspberryPi3/4 の方は、もうここで一緒に設定してしまいましょう。
　今回は、RaspberryPi2 WiFiなしモデルなので有効にしません。
- ロケール設定（設定はしなくてもいい）
　タイムゾーン、キーボードレイアウトを選択します
保存ボタンを押しましょう。
![](/images/20230611_030.png)
ここで、「書き込む」ボタンを押して、しばらく待ってください。
![](/images/20230611_040.png)
5分ぐらいで完了しました。
完了したら、そのままSDカードを抜いてOKです。
![](/images/20230611_050.png)


### 起動確認する
RaspberryPiにLANケーブルを接続（Wifiモデルなら不要）し、電源投入します。
そして、しばらく待ちます。

確認には、Macのターミナルを使います。
ターミナルを開いて、ネットワークに無事繋がったか、確認しましょう。

```
$ ping raspberrypi2
ping: cannot resolve raspberrypi2.local: Unknown host
```

Unknown hostと言われて失敗しました。
でも、焦らないでください。
ルーターからIPアドレスを取得し、ホスト名を登録するので、もう少し待ちましょう。
:::message
5分とか待ってもダメなら、ネットワーク／RaspberryPi起動周りから疑いましょう。
以下を確認してみてください。
- ルーターがRaspberryPIにDHCPでIPアドレスを付与しているか確認する
ルーターの設定ページなどで確認しましょう。
ちゃんと出来ていれば、IPアドレスが付与された、そのホスト名が何であるか表示されます。
- RaspberryPiの起動メッセージ確認する
RaspberryPiをHDMIケーブルでディスプレイに接続して、起動メッセージでエラーが出てたり、止まってないか確認しましょう。
:::

うまくいくと以下のような感じです。
ここでは、ホスト名 raspberrypi2 が 192.168.2.108 として見えています。
```
$ ping raspberrypi2      
PING raspberrypi2 (192.168.2.108): 56 data bytes
64 bytes from 192.168.2.108: icmp_seq=0 ttl=64 time=4.574 ms
64 bytes from 192.168.2.108: icmp_seq=1 ttl=64 time=5.905 ms
64 bytes from 192.168.2.108: icmp_seq=2 ttl=64 time=4.605 ms
^C
```

次は、ssh で接続してみましょう（ユーザー名は"pi"）。
パスワード聞かれたら、設定したパスワードを入力してください。
```
$ ssh pi@raspberrypi2
The authenticity of host 'raspberrypi2 (192.168.2.108)' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxx.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'raspberrypi2' (ED25519) to the list of known hosts.
pi@raspberrypi2's password: 
Linux raspberrypi2 6.1.21-v7+ #1642 SMP Mon Apr  3 17:20:52 BST 2023 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed May  3 09:37:13 2023

SSH is enabled and the default password for the 'pi' user has not been changed.
This is a security risk - please login as the 'pi' user and type 'passwd' to set a new password.

pi@raspberrypi2:~ $ 
```

Linux kernel 6.1のもので起動しているようです。
RaspberryPi、準備完了です。


## AWS IoT　を準備する
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
IoTデバイスとして、新しいモノを作成します。
名前は、"raspi2"とします。
![](/images/20230611_170.png)
#### Step3. プラットフォームとSDKを選択します
RaspberryPiのOSと、組み込むSDKの種類を選択します。
ここでは、Linux/macOS, Pythonを選びます。
![](/images/20230611_180.png)
#### Step4. 接続キットをダウンロード
接続キットをダウンロードします。
![](/images/20230611_190.png)
ダウンロードしたZIPファイルを、RaspberryPiに持っていきます。

Mac上で新しいターミナルを開いて、scp します。(パスワードを聞かれるので入力します)
```
＄ scp connect_device_package.zip pi@raspberrypi2:/home/pi
pi@raspberrypi2's password: 
connect_device_package.zip                    100% 6310     1.1MB/s   00:00  
```
成功したら、exit でターミナルを閉じます。
&nbsp;
続いて、RaspberryPiのターミナルで、scpしたZIPファイルを/home/pi/awsフォルダで解凍します。
```
pi@raspberrypi2:~ $ mkdir aws
pi@raspberrypi2:~ $ cp connect_device_package.zip aws
pi@raspberrypi2:~ $ cd aws
pi@raspberrypi2:aws $ unzip connect_device_package.zip
Archive:  connect_device_package.zip
 extracting: raspi2.cert.pem         
 extracting: raspi2.public.key       
 extracting: raspi2.private.key      
 extracting: raspi2-Policy           
 extracting: start.sh       
```
#### Step5. 接続キットを実行
解凍してできた start.sh に実行権を付与して、実行します。
```
pi@raspberrypi2:~ $ chmod +x start.sh
pi@raspberrypi2:~ $ ./start.sh

Downloading AWS IoT Root CA certificate from AWS...
<省略>

Cloning the AWS SDK...
Cloning into 'aws-iot-device-sdk-python-v2'...
remote: Enumerating objects: 2169, done.
remote: Counting objects: 100% (903/903), done.
remote: Compressing objects: 100% (360/360), done.
remote: Total 2169 (delta 681), reused 676 (delta 531), pack-reused 1266
Receiving objects: 100% (2169/2169), 2.09 MiB | 2.01 MiB/s, done.
Resolving deltas: 100% (1355/1355), done.

Installing AWS SDK...
Looking in indexes: https://pypi.org/simple, https://www.piwheels.org/simple
Processing ./aws-iot-device-sdk-python-v2
Collecting awscrt==0.16.19
  Downloading https://www.piwheels.org/simple/awscrt/awscrt-0.16.19-cp39-cp39-linux_armv7l.whl (7.1 MB)
     |████████████████████████████████| 7.1 MB 194 kB/s 
Building wheels for collected packages: awsiotsdk
  Building wheel for awsiotsdk (setup.py) ... done
  Created wheel for awsiotsdk: filename=awsiotsdk-1.0.0.dev0-py3-none-any.whl size=72344 sha256=0c10ec3cf5210740aead7ea964c4472182d0f841580e9e685d66c3e38f5bb140
  Stored in directory: /home/pi/.cache/pip/wheels/84/86/7c/cc52318eef66838b728ca3ef637a9432c10a0042fce6478e1f
Successfully built awsiotsdk
Installing collected packages: awscrt, awsiotsdk
Successfully installed awscrt-0.16.19 awsiotsdk-1.0.0.dev0
```
SDKをgit clone / Downloadして、インストールが成功しました。
そのまま放置すると、サンプルプログラムが動作し始めます。

```
Running pub/sub sample application...
Connecting to aqxxxx.ap-northeast-1.amazonaws.com with client ID 'basicPubSub'...
Connected!
Subscribing to topic 'sdk/test/python'...
Subscribed with QoS.AT_LEAST_ONCE
Sending messages until program killed
Publishing message to topic 'sdk/test/python': Hello World!  [1]
Received message from topic 'sdk/test/python': b'"Hello World!  [1]"'
Publishing message to topic 'sdk/test/python': Hello World!  [2]
Received message from topic 'sdk/test/python': b'"Hello World!  [2]"'
Publishing message to topic 'sdk/test/python': Hello World!  [3]
...
```

RaspberryPiのターミナルで、Hello Worldが延々と表示されていきます。
 
AWSのページも見てください。
![](/images/20230611_195.png)
右下に、"Hello World" というメッセージが順番に増えています。

そうです、
RaspberryPiから AWSに向かって、Hello　Worldを1秒に１回送っているのです。

おめでとうございます！
たったこれだけで、RaspberryPiとAWSは繋がるのです。

## 自分の好きなメッセージを送ってみる
・・・ですが、
最後は、いきなりゴールに行ってしまって、少し物足りないですよね。
もうちょっとだけ、調べてみましょう。

start.shを見てみましょう。
```bash:start.sh
#!/usr/bin/env bash
# stop script on error
set -e

# Check for python 3
if ! python3 --version &> /dev/null; then
  printf "\nERROR: python3 must be installed.\n"
  exit 1
fi

# Check to see if root CA file exists, download if not
if [ ! -f ./root-CA.crt ]; then
  printf "\nDownloading AWS IoT Root CA certificate from AWS...\n"
  curl https://www.amazontrust.com/repository/AmazonRootCA1.pem > root-CA.crt
fi

# Check to see if AWS Device SDK for Python exists, download if not
if [ ! -d ./aws-iot-device-sdk-python-v2 ]; then
  printf "\nCloning the AWS SDK...\n"
  git clone https://github.com/aws/aws-iot-device-sdk-python-v2.git --recursive
fi

# Check to see if AWS Device SDK for Python is already installed, install if not
if ! python3 -c "import awsiot" &> /dev/null; then
  printf "\nInstalling AWS SDK...\n"
  python3 -m pip install ./aws-iot-device-sdk-python-v2
  result=$?
  if [ $result -ne 0 ]; then
    printf "\nERROR: Failed to install SDK.\n"
    exit $result
  fi
fi

# run pub/sub sample app using certificates downloaded in package
printf "\nRunning pub/sub sample application...\n"
python3 aws-iot-device-sdk-python-v2/samples/pubsub.py --endpoint aqxxxx.ap-northeast-1.amazonaws.com --ca_file root-CA.crt --cert raspi2.cert.pem --key raspi2.private.key --client_id basicPubSub --topic sdk/test/python --count 0
```
コメントを頼りに見ていくと、最終行、pubsub.py が怪しいですね。
endpoint に向けて、topic sdk/test/python に対してメッセージを送っています。

aws-iot-device-sdk-python-v2/samples 下を見ると色々なコマンドが置いてありますので、
もう少し深掘りしたい方は見て頂きたいですが、
ひとまず、上記最終行の引数のcount を　1 に変更して、message を追加して保存します。
message は、”I am RaspberryPi" とします。

```diff bash
@@ -33,4 +33,4 @@
-python3 aws-iot-device-sdk-python-v2/samples/pubsub.py --endpoint aqxxxx.ap-northeast-1.amazonaws.com --ca_file root-CA.crt --cert raspi2.cert.pem --key raspi2.private.key --client_id basicPubSub --topic sdk/test/python --count 0
\ No newline at end of file
+python3 aws-iot-device-sdk-python-v2/samples/pubsub.py --endpoint aqxxxx.ap-northeast-1.amazonaws.com --ca_file root-CA.crt --cert raspi2.cert.pem --key raspi2.private.key --client_id basicPubSub --topic sdk/test/python --count 1 --message "I am RaspberryPi"
```

実行します。
```
pi@raspberrypi2:~/aws $ ./start.sh 

Running pub/sub sample application...
Connecting to aqxxxx.ap-northeast-1.amazonaws.com with client ID 'basicPubSub'...
Connected!
Subscribing to topic 'sdk/test/python'...
Subscribed with QoS.AT_LEAST_ONCE
Sending 1 message(s)
Publishing message to topic 'sdk/test/python': I am RaspberryPi [1]
Received message from topic 'sdk/test/python': b'"I am RaspberryPi [1]"'
1 message(s) received.
Disconnecting...
Disconnected!
```

好きなメッセージを１回だけ、送ることができました！
AWSのページでも確認できます！

これで、RaspberryPiから好きな時に、好きなメッセージを送ることができるようになりました。

第１回は以上です。
次回、第２回は、RaspberryPiをゼロからビルドできる環境を構築します。
引き続き、よろしくお願いします。
