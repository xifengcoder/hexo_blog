---
title: Https的理解
urlname: Https的理解
date: 2022-01-16 11:24:08
tags: Https
categories: Network
description: 通过WireShark抓包分析Https协议的内容...
---

通过Wireshark抓包来分析Https协议的内容。

#### 一、HTTPS协议详情

#####  1. Client Hello （Client -> Server）

客户端发送所支持的TLS协议版本、支持的密码套件列表、数据压缩方式及一个客户端随机数（client random）等信息发给服务端。

1）Handleshake Protocol 
2）TLS协议版本
3）户端生成的随机数client.random（32字节）；
4）支持的加密算法套件(Cipher Suites)，例如TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)
5） 支持的压缩方法
6）扩展属性

其中密码套件算法包含如下几部分：

1)  ECDHE：密钥交换算法（Elliptic Curve Diffie-Hellman Ephemeral，ECDHE）

2) RSA：认证算法（Rivest Shamir Adleman algorithm, RSA）

3) AES-128-GCM：数据加密算法（Advanced Encryption Standard 128 bit Galois/Counter Mode）

4) SHA256：消息摘要（MAC）算法（Secure Hash Algorithm 256 bit）

![Client Hello](/images/https_client_hello.jpg)

####  2.  Server Hello （Server --> Client ）

​    服务端在收到Client Hello消息后，发送服务端TLS协议版本、选择的密码套件（Cipher Suite）及服务器随机数（server random）等信息给客户端。服务器根据客户端传递的密码套件列表，选择一个双方都支持的密码套件进行处理。以TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)为例，表示为了协商出预备主密钥，需要使用ECDHE密钥协商算法，客户端和服务端每次连接的时候，服务端需要传递动态的DH信息（DH参数和DH公钥），传递的DH信息需要使用RSA签名算法签名后发送给客户端，相关细节会在Server Key Exchange子消息中说明。

包含内容:
1）Handleshake Protocol 

2）TLS协议版本

3)  服务器生成的随机数server.random（32字节）

4)  session ID(32字节)

5）选择的Cipher Suit

6) 扩展部分
![Server Hello](https://img-blog.csdnimg.cn/20190223210240623.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

#### 3.  Server Certificate （Server --> Client ）

服务端发送Certificate给客户端，Certificate包括以下几部分：

1）版本号(version): v3 (2)；

2）序列号(serialNumber):  长度16字节，例如：0x03aa20c7c44887f2b628b63d28a39e78；

3）签名算法(signature): sha256WithRSAEncryption，它包含一个Algorithm Id，例如1.2.840.113549.1.1.11，用来标识该签名算法；

4）版本者(issuer)；

5）有效期(validity)，包含起始时间(notBefore)和结束时间(notAfter)；

6）使用者(subject)

6.1 使用者公钥信息(subjectPublicKeyInfo)，包含以下内容：
   ① 算法类型(algorithm)，例如：rsaEncryption，其Algorithm Id是: 1.2.840.113549.1.1.1 (rsaEncryption)
   ② 公钥信息(subjectPublicKey)，包括公钥(modulus，长度256字节），指数(publicExponent，65537) 

6.2 算法标识(algorithmIdentifier)，例如：sha256WithRSAEncryption，具体含义是：

```
Public-Key Cryptography Standards (PKCS) #1 version 1.5 signature algorithm with Secure Hash Algorithm 256 (SHA256) and Rivest, Shamir and Adleman (RSA) encryption.
```

6.3 证书签名(encrypted)，长度256字节，例如：88291e3da5b2d39ed5406cb1185d27fd3abe233d3e4c2050...

![Server Certificate](https://img-blog.csdnimg.cn/20190223210317796.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

#### 4. Client Certificate (Client --> Server，可选)

服务端要求认证客户端的证书，这种情况使用的比较少。

#### 5. Server Key Exchange (Server --> Client)

如果服务端提供的证书不足以使客户端生成premaster key时，那么必须发送Server Key Exchange消息。

服务器发送动态或静态的DH信息（DH参数和DH公钥），使用服务器的私钥进行签名，这里选择的 SignatureAndHashAlgorithm 是rsa_pkcs1_sha256算法。

包含的内容如下：

- Content Type: Handshake (22) (1bytes)
- Version: TLS 1.2 (0x0303) (2bytes)

- Length: 333 (2bytes)

- Handshake Protocol: Server Key Exchange (333bytes)，包含：
      ① Handshake Type: Server Key Exchange (12) (3bytes)
  	② Length:329 (3bytes)
  	③ EC Diffie-Hellman Server Params(329bytes)，共有：
  		Curve Type: named_curve (0x03) (1bytes)
  		Named Curve: secp256r1 (0x0017) (2bytes)
  		Pubkey Length: 65 (1bytes)
  		Pubkey: 04f3f88d904af655faeeded2ba6453d34b1ba95160aaf427… (65bytes)
  		Signature Algorithm: rsa_pkcs1_sha256 (0x0401) (2bytes)
  		Signature Length: 256 (2bytes)
  		Signature: 4809b694a751868299023f90385b2012ed9a2916c957da16…（256bytes）

![Server Key Exchange](https://img-blog.csdnimg.cn/2019022321034489.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

#### 6. Server Hello Done (Server--> Client )

服务端确认Server Hello消息的结束。
![Server Hello Done](https://img-blog.csdnimg.cn/20190223210417283.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

#### 7. Client Key Exchange (Client --> Server)

客户端在收到Server Hello Done消息后会发送Client Key Exchange消息，如果服务端要求校验Client Certificate的话，Client Key Exchange消息在发送Client Certificate之后发送。

如果密码套件中的秘钥协商写法是ECDHE，那么客户端需要发送ECDH的公钥。

这里的重点是如何计算PreMaster Key。

- 若采用RSA， 客户端随机生成46字节+2字节的client_version，作为PreMaster Key，通过步骤（3）得到的证书中的公钥、或者Server Key Exchange消息中的临时RSA公钥，对其进行加密发送给服务器。
- 若采用DH算法（dhe_dss/dhe_rsa/dh_dss/dh_rsa/dh_anon等，下图就是DH算法生成PreMaster Key），直接根据已经交换的两个随机数可以算出PreMaster Key。

￼![Client Key Exchange](https://img-blog.csdnimg.cn/20190223210510907.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

#### 8. Client Change Cipher Spec (Client → Server)

表示客户端准备切换到加密环境，从这个消息后，客户端发送到的数据都将使用对称秘钥加密。

![Client Change Cipher Spec](https://img-blog.csdnimg.cn/20190223210528625.png)

#### 9. Client Encrypted handshake message （Client --> Server）

 Encrypted handshake message是使用对称秘钥加密后的第一个报文。 如果这个报文加解密校验成功，那么就说明对称秘钥是正确的。

![Client encrypted handshake message](https://img-blog.csdnimg.cn/20190223210536632.png)

#### 10. Server Change Cipher Spec (Server --> Client)

服务端开始切换加密环境。

服务端收到pre-master key之后，使用自己的Private Key来解密，服务端和客户端使用client random + server random + pre-master计算出master key（长度48bytes）。

```c
master_secret = PRF(pre_master_secret, "master secret", ClientHello.random + ServerHello.random) [0..47];
```
后面的消息就使用master key作为秘钥进行对称加密后在传输。

![Server change cipher cipher](https://img-blog.csdnimg.cn/20190223210612136.png)

#### 11. Server Encrypted handshake message (Server --> Client)

Encrypted HandShake Message是服务端使用对称秘钥加密后发送的第一条报文。

![Server Encrypted handshake message](https://img-blog.csdnimg.cn/20190223210621830.png)

#### 12. Application Data Protocol(Client -> Server)

客户端发给服务端的应用层数据，已使用对称秘钥加密。

![Client Application data protocol](https://img-blog.csdnimg.cn/20190223210644486.png)

#### 13. Application Data Protocol (Server--> Client)

服务端发给客户端的应用程层数据，已使用对称秘钥加密。

![Server Application data protocol](https://img-blog.csdnimg.cn/20190223210700490.png)

#### 14. Entrypted Alert(Server --> Client)

Alert（21）消息是一种通知消息（Close Notify），表示连接可以关闭了，通常用于数据已发送完成、没有数据要发送时。

![Encrypted Alert Message](https://img-blog.csdnimg.cn/20190223210720641.png)