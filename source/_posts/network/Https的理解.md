---
title: Https的理解
urlname: Https的理解
date: 2022-01-16 11:24:08
tags: Https
categories: Network
description: 通过WireShark抓包分析Https协议的内容...
---

﻿**一、对称密码**
**密钥配送问题：**

1. 通过实现共享密钥来解决
2. 通过密钥分配中心来解决
3. 通过Diffie-Hellman密钥交换来解决
4. 通过公钥密码来解决

**二、单项散列函数：**
又称消息摘要函数（Message Digest Function）、哈希函数或杂凑函数；散列值也称消息摘要（Message Digest）或者指纹（Fingerprint）。
**（1）性质**

1. 定长输出：根据任意长度的消息计算出固定长度的散列质；
2. 能够快速计算出散列值：计算散列值所花费的时间必须要短；

3. 抗碰撞性：哪怕只要1比特的改变，也必须有很高的概率产生不同的散列值；
       3.1 强抗碰撞性：要找到散列值相同的两条不同的消息时非常困难的；
       3.2 弱抗碰撞性：给定某条消息的散列值时，要找到和该条消息相同的散列值的另一条消息是非常困难的；
4. 单向性：无法通过散列值反算出原始消息；
   **（2）种类**
       1. MD5
       2. SHA-1、SHA-256、SHA-384、SHA-512
       3. RIPEMD-160
       4. AHS（Advanced Hash Standard）和SHA-3
          **（3）单向散列函数无法解决的问题**
           可以防篡改（确保完整性或一致性），但是无法辨别出“伪装”（无法认证），即无法确认消息来自正确的发送者，还是经第三方攻击者发出的。
          **三、消息认证码（Message Authentication Code，MAC）**
          **（1）理解：**
           消息认证码是一种与密钥相关的单向散列函数。
            (消息, 共享密钥)  ->  (MAC值)
          **（2）实现方法：**
          HMAC，例如HMAC-SHA1，HMAC-MD5，HMAC-SHA256.
          **（3）消息认证码无法解决的问题**
5. 对第三方证明
6. 防止否认
   **四、数字签名**
   **（1）包含**
7. 消息摘要算法
8. 签名算法：RSA和ECDSA，私钥加密，公钥解密。
   **（2）实际应用**
   先针对明文应用消息摘要算法生成散列值，然后应用签名算法进行加密生成签名信息。然后，将签名信息随明文一起发送出去。
   **（3）性质：**
    防篡改、防伪装、防止否认
   **（4）数字签名无法解决的问题：**
     无法保证公钥真正属于发送者。
   现在我们发现自己陷入了一个死循环——数字签名时用来识别消息篡改、伪装以及否认的，但是我们必须从没有被伪装的发送者得到没有被篡改的公钥才行。
   为此，我们需要使用证书。

**五、证书——为公钥加上数字签名**

（1）机密性：
依靠加密算法保障，例如AES-128-CBC；

加密算法可以保证机密性，但无法保证完整性。攻击者没有密钥，虽然无法破解数据，但是可以修改密文的部分数据，然后发送给接收方。接收方通过密钥发现可以解密，但是解出来的实际已不是原文，即消息已被篡改过。

（2）完整性：

消息验证码（Message Authentication Code，MAC）算法。MAC算法有两种形式，分别是：CMC-MAC算法和HMAC算法。例如HMAC-SHA256。

通信双方需维护一个密钥，只有拥有了密钥的双方才能生成和验证MAC值。


（3）机密性+完整性

1. 提供机密性和完整性的模式叫做Authentication Encryption（AE）模式，主要有：
   1）Encrypt-and-MAC
   2）MAC-then-Encrypt，HTTPS一般使用这种模式进行处理，例如AES-128-CBC#PKCS7-HMAC-SHA256；
   3）Encrypt-then-MAC，这种是比2更安全的模式。
2. AEAD加密模式
   AEAD（Authentication Encryption with Associated Data）模式在底层组合了加密算法和MAC算法，能够同时保证数据机密性和完整性。AEAD的作用类似Encrypt-then-MAC。
   主要有三种模式：
   1）CCM模式
   2）GCM模式，例如AES128-GCM，AES256-GCM；
   3）ChaCha20-Poly1305

一个Cipher Suites（加密算法套件）是以下4个算法的组合：

3. **Authentication（认证）算法**：有RSA，ECDSA，DH等；
4. **Encryption（加密）算法**：AES-128, AES-256, AESGCM256等；
5. **Message Authentication Code（消息认证, MAC）算法**：有AEAD, SHA1, SHA256等；
6. **Key Exchange（密钥交换）算法**，有ECDH, DH, DH/DSS，DH/RSA等。

### 一、HTTPS协议详情

####  1. Client Hello （Client -> Server）

客户端发送所支持的TLS协议版本、支持的密码套件列表、数据压缩方式及一个客户端随机数（client random）等信息发给服务端。

1）Handleshake Protocol 
2）TLS协议版本
3)  客户端生成的随机数client.random，长度48字节；
4) 支持的加密算法套件(Cipher Suites)，例如TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)
5) 支持的压缩方法
6）扩展属性

![1](https://img-blog.csdnimg.cn/20190223205741870.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)
![2](https://img-blog.csdnimg.cn/20190223205654335.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210001952.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210015609.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210023801.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

####  2.  Server Hello （Server --> Client ）

服务端在收到Client Hello消息后，发送服务端TLS协议版本、选择的密码套件（Cipher Suite）及服务器随机数（server random）等信息给客户端。

包含内容:
1）Handleshake Protocol 
2）TLS协议版本
3)  服务器生成的随机数server.random
4)  session ID(32字节)
5）选择的Cipher Suit，这里是：TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)
6) 扩展部分
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210240623.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

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

6.2 算法标识(algorithmIdentifier)，例如：sha256WithRSAEncryption

6.3 证书签名(encrypted)，长度256字节，例如：88291e3da5b2d39ed5406cb1185d27fd3abe233d3e4c2050...

![](https://img-blog.csdnimg.cn/20190223210317796.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

#### 4. Client Certificate (Client --> Server，可选)

服务端要求认证客户端的证书，这种情况使用的比较少。

#### 5. Server Key Exchange (Server --> Client)

如果服务端提供的证书不足以使客户端生成premaster key时，服务端会发送Server Key Exchange消息。

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

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019022321034489.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

#### 6. Server Hello Done (Server--> Client )

服务端确认Server Hello消息的结束。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210417283.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

#### 7. Client Key Exchange (Client --> Server)

客户端在收到Server Hello Done消息后会发送Client Key Exchange消息，如果服务端要求校验Client Certificate的话，Client Key Exchange消息在发送Client Certificate之后发送。

这里的重点是如何计算PreMaster Key。

- 若采用RSA， 客户端随机生成46字节+2字节的client_version，作为PreMaster Key，通过步骤（3）得到的证书中的公钥、或者Server Key Exchange消息中的临时RSA公钥，对其进行加密发送给服务器。
- 若采用DH算法（dhe_dss/dhe_rsa/dh_dss/dh_rsa/dh_anon等，下图就是DH算法生成PreMaster Key），字节根据已经交换的两个随机数可以算出PreMaster Key。

￼![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210510907.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpZmVucw==,size_16,color_FFFFFF,t_70)

#### 8. Client Change Cipher Spec (Client → Server)

表示客户端准备切换到加密环境，从这个消息后，客户端发送到的数据都将使用对称秘钥加密。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210528625.png)

#### 9. Encrypted handshake message （Client --> Server）

 Encrypted handshake message是使用对称秘钥加密后的第一个报文。 如果这个报文加解密校验成功，那么就说明对称秘钥是正确的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210536632.png)

#### 10. Server Change Cipher Spec (Server --> Client)

服务端开始切换加密环境。

服务端收到pre-master key之后，使用自己的Private Key来解密，服务端和客户端使用client random + server random + pre-master计算出master key（长度48bytes）。

```c
master_secret = PRF(pre_master_secret, "master secret", ClientHello.random + ServerHello.random) [0..47];
```
后面的消息就使用master key作为秘钥进行对称加密后在传输。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210612136.png)

#### 11. Encrypted handshake message (Server --> Client)

Encrypted HandShake Message是服务端使用对称秘钥加密后发送的第一条报文。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210621830.png)

#### 12. Application Data Protocol(Client -> Server)

客户端发给服务端的应用层数据，已使用对称秘钥加密。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210644486.png)

#### 13. Application Data Protocol (Server--> Client)

服务端发给客户端的应用程层数据，已使用对称秘钥加密。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210700490.png)

#### 14. Entrypted Alert(Server --> Client)

Alert（21）消息是一种通知消息（Close Notify），表示连接可以关闭了，通常用于数据已发送完成、没有数据要发送时。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223210720641.png)