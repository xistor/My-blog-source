---
title: "IKEV2 vpn 配置备忘"
date: 2020-05-27T23:03:16+08:00
tags: ["vpn"]
categories: ["network"]
---

## 1.

首先用这位同学的一键脚本完成安装及部分配置工作：https://github.com/quericy/one-key-ikev2-vpn

## 2.

配置/usr/local/etc/ipsec.secrets, 添加用户

```
: RSA server.pem
: PSK "yourpskxxx"
: XAUTH "yourxauthxxx"
xx : XAUTH "xxxxx"
xxx : XAUTH "xxxxx"
```

## 3.
若登录不成功, 查看log

```
tail -100 /var/log/syslog
```

若有类似如下log:

```
charon: 14[CFG] received proposals: IKE:AES_CBC_128/HMAC_SHA1_96/PRF_HMAC_SHA1/MODP_1024
charon: 14[CFG] configured proposals: IKE:AES_CBC_256/HMAC_SHA1_96/PRF_HMAC_SHA1/MODP_1024
charon: 14[IKE] no proposal found
charon: 14[ENC] generating INFORMATIONAL_V1 request 3851683074 [ N(NO_PROP) ]

```

原因是hash算法不匹配，在`/usr/local/etc/ipsec.conf` 里的 `iOS_cert` 和 `android_xauth_psk` 添加`ike`项，指定hash算法。

```
config setup
    uniqueids=never 

conn iOS_cert
    keyexchange=ikev1
    ike=aes256-sha256-modp1024,3des-sha1-modp2048,aes256-sha1-modp2048!
    fragmentation=yes
    left=%defaultroute
    leftauth=pubkey
    leftsubnet=0.0.0.0/0
    leftcert=server.cert.pem
    right=%any
    rightauth=pubkey
    rightauth2=xauth
    rightsourceip=10.31.2.0/24
    rightcert=client.cert.pem
    auto=add

conn android_xauth_psk
    keyexchange=ikev1
    ike=aes256-sha256-modp1024,3des-sha1-modp2048,aes256-sha1-modp2048!
    left=%defaultroute
    leftauth=psk
    leftsubnet=0.0.0.0/0
    right=%any
    rightauth=psk
    rightauth2=xauth
    rightsourceip=10.31.2.0/24
    auto=add

...

```

### 4.

android 客户端登录 

vpn类型选 `Ipsec Xauth PSK` 
Ipsec预分享密钥 填`ipsec.secrets`中的PSK 
