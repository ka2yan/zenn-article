---
title: "AWS IoT に必要な機能を追加する"
---

RaspberryPiにAWS IoT接続に必要な機能を組み込みます。
本章では、AWS IoT接続に必要な証明書や鍵情報は、AWSで作成したものを静的に組み込みます。
証明書や鍵を後から動的に組み込むことは、次章以降でトライしたいと思います。

## meta-aws とは
AWSと通信するためのコマンドやライブラリを組み込むためのYoctoプロジェクト向けのBSPレイヤー（レシピ集）です。
[meta-aws公式ページ](https://github.com/aws4embeddedlinux/meta-aws)


## Greengrass とは
AWS Greengrassとは、xxx

組み込み方は、meta-awsの以下のページに記載があります。
[Greengrass説明ページ](https://github.com/aws4embeddedlinux/meta-aws/tree/master/recipes-iot/aws-iot-greengrass)

この２ページを参考に組み込みましょう。

```
$ cd poky
$ git clone https://github.com/openembedded/meta-openembedded
$ git clone https://github.com/aws4embeddedlinux/meta-aws
```

クローンしたBSPレイヤーをビルド対象に追加します。
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

IoT Greengrass デモ用バイナリーを組み込みます。
GGV2_DATA_EPなどは後で設定するので、ここでは空値でOKです。
```diff bash:build/conf/local.conf
@@ -281,3 +281,15 @@
 # track the version of this file when it was generated. This can safely be ignored if
 # this doesn't mean anything to you.
 CONF_VERSION = "2"
+
+# systemd
+DISTRO_FEATURES += "systemd"
+DISTRO_FEATURES_BACKFILL_CONSIDERED = "sysvinit"
+VIRTUAL-RUNTIME_init_manager = "systemd"
+VIRTUAL-RUNTIME_initscripts = ""
+
+# AWS IoT Greengrass demo
+IMAGE_INSTALL:append = " greengrass-bin-demo"
+
+GGV2_DATA_EP     = ""
+GGV2_CRED_EP     = ""
+GGV2_REGION      = ""
+GGV2_PKEY        = ""
+GGV2_CERT        = ""
+GGV2_CA          = ""
+GGV2_THING_NAME  = ""
+GGV2_TES_RALIAS  = ""
```

SSHも使えるようにしておきましょう。
```diff bash:build/conf/local.conf
-293,3 +293,6 @@
+IMAGE_INSTALL:append = " openssh openssh-sftp-server"
```

ビルドする
追加されたBSPレイヤーなどみて、再度OSS間の依存関係の解析が走ります。。。
```
$ bitbake core-image-minimal
Loading cache: 100% |                                     | ETA:  --:--:--
Loaded 0 entries from dependency cache.
Parsing recipes:  82% |##############################     | Time: 
```

続いて、ビルドが開始されました。
```
...
Parsing recipes: 100% |#####################################| Time: 0:01:24
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

Initialising tasks: 100% |#################################| Time: 0:00:01
Sstate summary: Wanted 190 Local 0 Mirrors 0 Missed 190 Current 1477 (0% match, 88% complete)
Removing 4 stale sstate objects for arch raspberrypi2: 100% |########| Time: 0:00:00
NOTE: Executing Tasks
Setscene tasks: 1668 of 1668
Currently  4 running tasks (3159 of 3777)  83% |##############            |
0: sudo-1.9.13p3-r0 do_compile - 51s (pid 43711)
1: libx11-1_1.8.5-r0 do_compile - 51s (pid 43839)
2: xorgproto-2022.2-r0 do_package_write_rpm - 0s (pid 63352)
3: xorgproto-2022.2-r0 do_package_qa - 0s (pid 63362)
```
前回は3408個のOSSをビルドしましたが、今回は3777個に増えました。
今回もビルドが完了するまで待ちましょう。

終わりました。

前回同様、
Mac側のターミナルから、"docker cp" コマンドでイメージファイルをとってきます。

```
$ docker ps
CONTAINER ID   IMAGE          COMMAND       CREATED      STATUS              PORTS     NAMES
b7c4cb3eeb5c   ubuntu:22.04   "/bin/bash"   1 days ago   Up About 10 hour 
$
$ docker cp b7c4cb3eeb5c:/home/hoge/raspi2/poky/build/tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2-20230701141645.rootfs.wic.bz2 .
                              Successfully copied 232MB to /XXX/XXX/.
$
$ bunzip2 core-image-minimal-raspberrypi2-20230701141645.rootfs.wic.bz2
$ ls -lh
total 1311656
-rw-r--r--@ 1 katsu  staff  633M  7  1 23:22 core-image-minimal-raspberrypi2-20230701141645.rootfs.wic
```
今回は、633MBのファイルができました。

## 焼いて起動する

Greengrass バージョン確認する
```
# java -jar /greengrass/v2/alts/init/distro/lib/Greengrass.jar --version
AWS Greengrass v2.9.6
```

## AWS IoT 設定する
AWS Consoleにログインする。

CloudShellを開く

Data-ATS Endpoint情報を取得する
```
＄ aws iot describe-endpoint --endpoint-type iot:Data-ATS
{
    "endpointAddress": "aqt2g7hn14qrk-ats.iot.ap-northeast-1.amazonaws.com"
}
```

CredentialProvide endpoit情報を取得する
```
$ aws iot describe-endpoint --endpoint-type iot:CredentialProvider
{
    "endpointAddress": "c5miqvgs8zo55.credentials.iot.ap-northeast-1.amazonaws.com"
}
```

IoTデバイス（Thing） を作ります。
名前は、”RaspiGreengrassCore"
```
$ aws iot create-thing --thing-name RaspiGreengrassCore
{
    "thingName": "RaspiGreengrassCore",
    "thingArn": "arn:aws:iot:ap-northeast-1:004996099279:thing/RaspiGreengrassCore",
    "thingId": "0e09a46d-3134-451b-ba0c-61b866400589"
}
```

IoTデバイスグループを作ります。
名前は、"RaspiGreengrassCoreGroup"
```
＄ aws iot create-thing-group --thing-group-name RaspiGreengrassCoreGroup
{
    "thingGroupName": "RaspiGreengrassCoreGroup",
    "thingGroupArn": "arn:aws:iot:ap-northeast-1:004996099279:thinggroup/RaspiGreengrassCoreGroup",
    "thingGroupId": "d1353502-5f71-4f51-b56f-43d057aec3b0"
}
```

作成したグループにIoTデバイスを追加します。
```
$ aws iot add-thing-to-thing-group --thing-name RaspiGreengrassCore --thing-group-name RaspiGreengrassCoreGroup
```


### Raspi用証明書を作る
RaspberryPiに組み込むクライアント証明書、公開鍵、秘密鍵を作ります。
```
$ aws iot create-keys-and-certificate --set-as-active --certificate-pem-outfile device.pem.crt --public-key-outfile public.pem.key --private-key-outfile private.pem.key
{
    "certificateArn": "arn:aws:iot:ap-northeast-1:004996099279:cert/327444387d11f1a9c260b8fc0e6172d40e8528b33a942bcfdef732387e4f11b0",
    "certificateId": "327444387d11f1a9c260b8fc0e6172d40e8528b33a942bcfdef732387e4f11b0",
    "certificatePem": "-----BEGIN CERTIFICATE-----\nMIIDWjCCAkKgAwIBAgIVALqVMa8lMgCajUqdBHKk5kDWndkiMA0GCSqGSIb3DQEB\nCwUAME0xSzBJBgNVBAsMQkFtYXpvbiBXZWIgU2VydmljZXMgTz1BbWF6b24uY29t\nIEluYy4gTD1TZWF0dGxlIFNUPVdhc2hpbmd0b24gQz1VUzAeFw0yMzA3MTIwOTA4\nMTFaFw00OTEyMzEyMzU5NTlaMB4xHDAaBgNVBAMME0FXUyBJb1QgQ2VydGlmaWNh\ndGUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDdlZ+pNyFVqsCBjGGo\nuFL+ZbArm96cb6mlHFETRAsKiPOU2TKawc8Lmdj5HuyItQufullIhbACu+3/Rj+b\nK8F1YJIsux86LwPesLn4to34EA5/Hr+shHQ7UYMRTMSqx+wHeT5wFpoEeume5SOo\nN5IocwXADVfweKaSGi4sR2idhlT7zcBtACml/vT9kcUQdeKbtc4m2ajJb5wAItp/\nIByuZPGbycH7Q+O9nLr5ED0dicTj7hds6uOR0IMfkCtiUhXQ4SXGTS+eDVxrSmFF\nBGv908ChowHjGTDFNNwT4QMsWTNJm08da7n89Y3GbiTe0mn85TFLSXD7cBhLrTlX\nmosNAgMBAAGjYDBeMB8GA1UdIwQYMBaAFHc29RXDbpLJbBoTWt7t3ncdfkX8MB0G\nA1UdDgQWBBR7N6eiUWjnmFPsTCeeI1t8bGESyzAMBgNVHRMBAf8EAjAAMA4GA1Ud\nDwEB/wQEAwIHgDANBgkqhkiG9w0BAQsFAAOCAQEAqxyRJNKz1w1gKZbCZLdOaDlE\njsF3+NjZs+SZObgSU/zatyYHFpJQYKtsPU6eDDSTkGuIsUqvda4JVFj7Ro75BOFS\n3rW4RUlRlYJ8onKuR1CXchmgouMqTREdt5tM4gdF/hV0i0QdciJrZEhk12uEqB1p\n8iqj+3B6VwysMBYTmQSunX3RBSFwsmjWMvR/W2778Zp4mDboFc55VT4AKP9o/JC+\n0rJzjBzDJJUs3PgmwHFvnlxB1/imR1j7SEBy56qgiCzPOhLwNO8UT1YJNDcDPYpn\nR0vC75k9GJl9YhcOA6WuCE1HlqX9slTxXAzZqxl/Jo8zrfUl+0R82eZYsNSdJQ==\n-----END CERTIFICATE-----\n",
    "keyPair": {
        "PublicKey": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA3ZWfqTchVarAgYxhqLhS\n/mWwK5venG+ppRxRE0QLCojzlNkymsHPC5nY+R7siLULn7pZSIWwArvt/0Y/myvB\ndWCSLLsfOi8D3rC5+LaN+BAOfx6/rIR0O1GDEUzEqsfsB3k+cBaaBHrpnuUjqDeS\nKHMFwA1X8HimkhouLEdonYZU+83AbQAppf70/ZHFEHXim7XOJtmoyW+cACLafyAc\nrmTxm8nB+0PjvZy6+RA9HYnE4+4XbOrjkdCDH5ArYlIV0OElxk0vng1ca0phRQRr\n/dPAoaMB4xkwxTTcE+EDLFkzSZtPHWu5/PWNxm4k3tJp/OUxS0lw+3AYS605V5qL\nDQIDAQAB\n-----END PUBLIC KEY-----\n",
        "PrivateKey": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpQIBAAKCAQEA3ZWfqTchVarAgYxhqLhS/mWwK5venG+ppRxRE0QLCojzlNky\nmsHPC5nY+R7siLULn7pZSIWwArvt/0Y/myvBdWCSLLsfOi8D3rC5+LaN+BAOfx6/\nrIR0O1GDEUzEqsfsB3k+cBaaBHrpnuUjqDeSKHMFwA1X8HimkhouLEdonYZU+83A\nbQAppf70/ZHFEHXim7XOJtmoyW+cACLafyAcrmTxm8nB+0PjvZy6+RA9HYnE4+4X\nbOrjkdCDH5ArYlIV0OElxk0vng1ca0phRQRr/dPAoaMB4xkwxTTcE+EDLFkzSZtP\nHWu5/PWNxm4k3tJp/OUxS0lw+3AYS605V5qLDQIDAQABAoIBAQCBiP6VRY1PL0rq\ncM6Ge3rJDVk3pR82BHD//NXIlXZ+6iC7W12h6rrG5WFaASH1qSDqd13Kb5y9fG9d\nVAvLAoFNxO6vB5TxxppUjKurIc1MvtY6qhcTGzt3kec1LdOqosTweYhurkfLZq88\nHGgD5riivNsXsrU99sopjvR/Hh+iNdbrN3xdDU2Ewlo/atMO93b6FCghcEqSGaWh\nLyJRqwOMsM6FUKzRol6oYcu6GVmer/fd8IxbsnPLmsNEM1Br9yUVMX+qSS1b3Pr1\nh1Pr1WJm2ZyhyRRueme1HsPbskbv9iNmQnb0eHObBG7kJI7hgCu5AU7gVLn1d3G+\nYBXepSkBAoGBAPEsOGhSTPqod23DghFsRlGCtwhplZCUkjkH4/M1HU+YfeRc0A78\nO3zxKcJJq4YoyrWe8FkzACkRSs+T/0Tt4i8w2NGhWILPLBUJb+gJK37a7b9KlW7o\nkOaE6vJ93MR3tRZ7X5kG/bhiFH8QSSeEnFnJ7z2Mt7y6ycNJc7oJoNzFAoGBAOs1\nG06qbnnC46OnSopBKzJ4ckaVffXfM0q8PIcrjxZ7fjQRRqlOJyL/jMNdF0koUhKm\ntKdRmxUtFlxCO+2Ru9rIS1RwlxOEPXHPNXIaYBMPybQuOmHNXrapqQ6q+9W0EpVR\nsgcUkA6gArjN2xfu0yKUe+nKaYshXmLC67DQLGmpAoGBAKASA5ZqGaG8sxftTaUW\nwk1TfvxcZ+LAWZT0wb0oob20rsolOAraKvmwb1D+6JNw+6o0Rb5OdWrMiWThC+rK\nIPfFagMpHcAklVOZIedWPsJBuM7gR/KG9bWqvu4Xz7Gu6khztm2xEDGTF5uGSaer\nAsMtnlax0Tm4mDW/yMnPni8pAoGADbArhKp6f2+OG+oSdnVQdEF6NQ1iJTr2GzVV\nOHCahS5uq80NlbDMqkbBBGWYg1NrY1Z8UPh41ASptnjMUAkZK6RYbfOXdzVM9iCe\n9aL/UFys2mWOVD7Fck/xXL8qpMc0BaiZebwCnjdFsUeZpozpkKufgn2bItOwUIMT\ngFi9HPECgYEA3lm63gxhE2VTxNX/2wwXGei1BQaC1Z2SP7yzBS6P7rYVahpPqrkx\nFFI68E1qiTaSOII3+h7TwOh+OefhqxcInx1UHrsE40AJy9drP7rWQP9V2FlW0Ezb\n2Eg85HQrIfBt2ZPJJtyWOHL8fb197K/cWPKCjwTwRwdSNIFXKexTD2Y=\n-----END RSA PRIVATE KEY-----\n"
    }
}
```
１行目の certificateArn を覚えておいてください。

作成した３ファイルは、ダウンロードしておきましょう。
「アクション」から「ファイルのダウンロード」を選択し、
ファイル名を指定して、ダウンロードしてください。
（フォルダ指定ではダウンロードできないようです）

作成したcertificateArnを、"RaspiGreengrassCore"の証明書として紐付け（アタッチ）します。
```
$ aws iot attach-thing-principal --thing-name RaspiGreengrassCore --principal arn:aws:iot:ap-northeast-1:004996099279:cert/327444387d11f1a9c260b8fc0e6172d40e8528b33a942bcfdef732387e4f11b0
```

### IoTポリシーを作る
IoTポリシーファイルを作りましょう。
ファイル名は、”greengrass-v2-iot-policy.json"とします。

```json: greengrass-v2-iot-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iot:Publish",
        "iot:Subscribe",
        "iot:Receive",
        "iot:Connect",
        "greengrass:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

"GreengrassV2IoTThingPolicy"というポリシーを作成します。
```
$ aws iot create-policy --policy-name GreengrassV2IoTThingPolicy --policy-document file://greengrass-v2-iot-policy.json
 
{
    "policyName": "GreengrassV2IoTThingPolicy",
    "policyArn": "arn:aws:iot:ap-northeast-1:004996099279:policy/GreengrassV2IoTThingPolicy",
    "policyDocument": "{\n  \"Version\": \"2012-10-17\",\n  \"Statement\": [\n     {\n     \t\"Effect\": \"Allow\",\n        \"Action\": [\n            \"iot:Publish\",\n            \"iot:Subscribe\",\n            \"iot:Receive\",\n            \"iot:Connect\",\n            \"greengrass:*\"\n        ],\n        \"Resource\": [\n            \"*\"\n\t]\n     }\n  ]\n}\n",
    "policyVersionId": "1"
}
```

Raspi用証明書（certificateArn）と、IoTポリシーの紐付け（アタッチ）します。
```
$ aws iot attach-policy --policy-name GreengrassV2IoTThingPolicy --target arn:aws:iot:ap-northeast-1:004996099279:cert/327444387d11f1a9c260b8fc0e6172d40e8528b33a942bcfdef732387e4f11b0
```


### Roleを作る
Trust Roleファイルを作成します。
```json:device-role-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "credentials.iot.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

"GreengrassV2TokenExchangeRole"というRoleを作成します。
```
$ aws iam create-role --role-name GreengrassV2TokenExchangeRole --assume-role-policy-document file://device-role-trust-policy.json
{
    "Role": {
        "Path": "/",
        "RoleName": "GreengrassV2TokenExchangeRole",
        "RoleId": "AROAQCKOKNTH4JPXOCIGR",
        "Arn": "arn:aws:iam::004996099279:role/GreengrassV2TokenExchangeRole",
        "CreateDate": "2023-07-12T10:00:05+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "credentials.iot.amazonaws.com"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}
```
作成したRole ARNを覚えておきましょう。

### アクセスポリシーを作る
同様にアクセスポリシーファイルを作成します。
```json:device-role-access-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    }
  ]
}
```

”GreengrassV2TokenExchangeRoleAccess"というポリシーを作成します。
```
$ aws iam create-policy --policy-name GreengrassV2TokenExchangeRoleAccess --policy-document file://device-role-access-policy.json
{
    "Policy": {
        "PolicyName": "GreengrassV2TokenExchangeRoleAccess",
        "PolicyId": "ANPAQCKOKNTH5BAXRXCUS",
        "Arn": "arn:aws:iam::004996099279:policy/GreengrassV2TokenExchangeRoleAccess",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2023-07-12T10:07:06+00:00",
        "UpdateDate": "2023-07-12T10:07:06+00:00"
    }
}
```

ロール名"GreengrassV2TokenExchangeRole"を付けます。
```
$ aws iam attach-role-policy --role-name GreengrassV2TokenExchangeRole --policy-arn arn:aws:iam::004996099279:policy/GreengrassV2TokenExchangeRoleAccess
```

作成したRole ARNと紐付けます。
```
$ aws iot create-role-alias --role-alias GreengrassCoreTokenExchangeRoleAlias --role-arn arn:aws:iam::004996099279:role/GreengrassV2TokenExchangeRole
{
    "roleAlias": "GreengrassCoreTokenExchangeRoleAlias",
    "roleAliasArn": "arn:aws:iot:ap-northeast-1:004996099279:rolealias/GreengrassCoreTokenExchangeRoleAlias"
}
```

### IoT Role Alias を作る
```json:greengrass-v2-iot-role-alias-policy.json 
{
  "Version":"2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:AssumeRoleWithCertificate",
      "Resource": "arn:aws:iot:ap-northeast-1:004996099279:rolealias/GreengrassCoreTokenExchangeRoleAlias"
    }
  ]
}
```

```
$ aws iot create-policy --policy-name GreengrassCoreTokenExchangeRoleAliasPolicy --policy-document file://greengrass-v2-iot-role-alias-policy.json
{
    "policyName": "GreengrassCoreTokenExchangeRoleAliasPolicy",
    "policyArn": "arn:aws:iot:ap-northeast-1:004996099279:policy/GreengrassCoreTokenExchangeRoleAliasPolicy",
    "policyDocument": "{\n  \"Version\":\"2012-10-17\",\n  \"Statement\": [\n    {\n      \"Effect\": \"Allow\",\n      \"Action\": \"iot:AssumeRoleWithCertificate\",\n      \"Resource\": \"arn:aws:iot:ap-northeast-1:004996099279:rolealias/GreengrassCoreTokenExchangeRoleAlias\"\n    }\n  ]\n}\n",
    "policyVersionId": "1"
}
```

IoT Thing ARN と IoT Role Aliasを紐付けます。
```
$ aws iot attach-policy --policy-name GreengrassCoreTokenExchangeRoleAliasPolicy --target arn:aws:iot:ap-northeast-1:004996099279:cert/327444387d11f1a9c260b8fc0e6172d40e8528b33a942bcfdef732387e4f11b0
```

長い旅でした。
AWS側の準備は一旦これで終わりです。


## RaspberryPiにAWS情報を組み込む
RaspberryPiのビルドに戻ります。
AWS Consoleで作成した情報を、Yocto上の local.conf　に記載します。

### AWS 接続先情報を指定する
まず、AWS CLI上で出てきた情報を整理しましょう。

- Data-ATS Endpoint情報
aqt2g7hn14qrk-ats.iot.ap-northeast-1.amazonaws.com

- CredentialProvide endpoit情報を取得する
c5miqvgs8zo55.credentials.iot.ap-northeast-1.amazonaws.com

- Region
AWS CloudShell のタブ名 ”ap-northeast-1"です。
コマンドで確認する場合は、以下のAWS CLIコマンドで確認してください。
```
$ aws configure get region
または、
$ echo $AWS_DEFAULT_REGION
```

- Thing name
RaspiGreengrassCore

- Token Exchange Role Alias
GreengrassCoreTokenExchangeRoleAlias


５つの項目を以下に記載します。
```diff bash:build/conf/local.conf
@@ -285,14 +285,14 @@
-GGV2_DATA_EP     = ""
-GGV2_CRED_EP     = ""
-GGV2_REGION      = ""
+GGV2_DATA_EP     = "aqt2g7hn14qrk-ats.iot.ap-northeast-1.amazonaws.com"
+GGV2_CRED_EP     = "c5miqvgs8zo55.credentials.iot.ap-northeast-1.amazonaws.com"
+GGV2_REGION      = "ap-northeast-1"
 GGV2_PKEY        = ""
 GGV2_CERT        = ""
 GGV2_CA          = ""
-GGV2_THING_NAME  = ""
-GGV2_TES_RALIAS  = ""
+GGV2_THING_NAME  = "RaspiGreengrassCore"
+GGV2_TES_RALIAS  = "GreengrassCoreTokenExchangeRoleAlias"```
```

### 証明書を上書きする

Macのターミナルから、AmazonrルートCA証明書をダウンロードします。
AWSがWebサイトで公開しているファイルなので、curlコマンドで取ってきます。
```
$ cd Downloads
$ curl -o ./AmazonRootCA1.pem https://www.amazontrust.com/repository/AmazonRootCA1.pem
```
続いて、"docker cp"コマンドでクライアント証明書、秘密鍵、ルートCA証明書をUbuntu環境に持っていきます。
（公開鍵は、すみません、使用しません）
```
$ cd Downloads
$ docker cp device.pem.crt b7c4cb3eeb5c:/home/hoge/raspi2/poky/meta-aws/recipes-iot/aws-iot-greengrass/greengrass-bin-demo
$ docker cp private.pem.key b7c4cb3eeb5c:/home/hoge/raspi2/poky/meta-aws/recipes-iot/aws-iot-greengrass/greengrass-bin-demo
$ 
$ docker cp AmazonRootCA1.pem b7c4cb3eeb5c:/home/hoge/raspi2/poky/meta-aws/recipes-iot/aws-iot-greengrass/greengrass-bin-demo
```

Ubuntuのターミナルから、クライアント証明書、秘密鍵、ルートCA証明書を更新します。
Docker consoleから、
```
$ cd home/hoge/raspi2/poky/meta-aws/recipes-iot/aws-iot-greengrass/greengrass-bin-demo
$ mv device.pem.crt demo.cert.pem 
$ mv private.pem.key demo.pkey.pem 
$ mv AmazonRootCA1.pem demo.root.pem 
```

ついに、ビルドします。
```
$ bitbake core-image-minimal
Loading cache: 100% |                            | ETA:  --:--:--
Loaded 0 entries from dependency cache.
Parsing recipes: 100% |##########################| Time: 0:01:16
Parsing of 2751 .bb files complete (0 cached, 2751 parsed). 4507 targets, 171 skipped, 0 masked, 0 errors.
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
meta-oe              
meta-multimedia      
meta-networking      
meta-python          = "master:bf314d2c57d63afd680ab21baf43e150456120ed"
meta-aws             = "master:c370b087fa54a2b2a01c8ecff6d4d05ea9d141bf"

Initialising tasks: 100% |#######################| Time: 0:00:01
Sstate summary: Wanted 1675 Local 1671 Mirrors 0 Missed 4 Current 0 (99% match, 0% complete)
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 3796 tasks of which 3783 didn't need to be rerun and all succeeded.
```
完成しました。
イメージファイル名を確認します。
```
$ ls tmp/deploy/images/raspberrypi2/*.wic.bz2
tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2-20230713140331.rootfs.wic.bz2
tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2.wic.bz2
```

Macのターミナルから、”docker cp"コマンドで取得します。
```
$ docker ps
    COMMAND       CREATED       STATUS       PORTS     NAMES
b7c4cb3eeb5c   ubuntu:22.04   "/bin/bash"   2 weeks ago   Up 13 days             raspi2-env
$ docker cp b7c4cb3eeb5c:/home/hoge/raspi2/poky/build/tmp/deploy/images/raspberrypi2/core-image-minimal-raspberrypi2-20230713140331.rootfs.wic.bz2 .
$ bunzip2 core-image-minimal-raspberrypi2-20230713140331.rootfs.wic.bz2
$ ls -l 
total 4099760
-rw-r--r--@ 1 katsu  staff  663957504  7 13 23:05 core-image-minimal-raspberrypi2-20230713140331.rootfs.wic
```

### RaspberryPiに焼いてみる
書き込みに、前章で使用した RaspberryPi Imager を使用しましょう。
カスタームイメージを使う、で、

動かしてみる？
