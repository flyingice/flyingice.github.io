---
title: RSA密钥格式
author: Yibin Yang
tag: ssh openssl
---

SSH几乎天天都在用，但是我从来没想过密钥到底是如何存储的，刚好这几天倒腾GPG密钥的时候碰到点配置问题，于是把和RSA密钥相关的东西整理了一下。

首先ssh-keygen和openssl都可以用来生成密钥对，ssh-keygen背后也是通过调用OpenSSL库来工作的，所以下面的demo都以openssl CLI为例（环境为MacOS 10.15.5 + LibreSSL 2.8.3）。但是ssh-keygen和openssl生成的公钥格式在默认情况下是不同的，后文具体说明。

**如何判断私钥是否被加密**

先拿openssl生成一个2048位的未加密私钥，再另外建一个加密过的私钥，对应passphrase是12345 。

```
> openssl genrsa -out key.clear 2048
Generating RSA private key, 2048 bit long modulus
............+++
.......................................................................................................+++
e is 65537 (0x10001)

> cat key.clear
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

```
> openssl genrsa -aes256 -out key.encrypted -passout pass:12345 2048
Generating RSA private key, 2048 bit long modulus
...........+++
....................................................................................+++
e is 65537 (0x10001)

> cat key.encrypted
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-256-CBC,7290859FA04F937A74CD4C1540049234

...
-----END RSA PRIVATE KEY-----
```

第二个openssl genrsa命令中的-aes256指定了用于加密私钥的对称加密算法，具体密钥由KDF基于passphrase生成。对比一下输出文件，可以看到经过加密的版本多了两行文件头，其中AES-256-CBC是我们刚才在命令行指定的对称加密算法，之后紧接的是一串hex编码的IV，它在CBC模式下加密第一个数据块时使用。CBC模式保证了在不同位置出现的相同数据能生成不同的密文，从而提高了安全性。

**RSA私钥是什么格式**

RSA私钥通常用PKCS#1或者PKCS#8格式存储。信息以ASN.1数据结构为载体，DER序列化成二进制后用base64编码。ASN.1有些类似json和protobuf，支持string和integer等数据类型，同时也能多层嵌套形成树形结构。openssl有专门的工具包asn1parse来解析ASN.1数据（如下所示）。

```
> openssl rsa -in key.encrypted -passin pass:12345 -out key.decrypted
> openssl asn1parse -in key.decrypted
0:d=0  hl=4 l=1188 cons: SEQUENCE
    4:d=1  hl=2 l=   1 prim: INTEGER           :00
    7:d=1  hl=4 l= 257 prim: INTEGER           :......
  268:d=1  hl=2 l=   3 prim: INTEGER           :010001
  273:d=1  hl=4 l= 257 prim: INTEGER           :......
  534:d=1  hl=3 l= 129 prim: INTEGER           :......
  666:d=1  hl=3 l= 129 prim: INTEGER           :......
  798:d=1  hl=3 l= 129 prim: INTEGER           :......
  930:d=1  hl=3 l= 128 prim: INTEGER           :......
 1061:d=1  hl=3 l= 128 prim: INTEGER           :......
```

以上结果略去了大部分integer值。前四个整数依次是版本号，e（public exponent），模数n以及d（private exponent）。这里顺便回顾一下生成RSA密钥的原理：

1）任意取两个超大质数p和q，求得n=p * q，z=(p-1) * (q-1)。 

2）取一个小于n且与z互质的数，记作e。这里的e就是public exponent，数学上要求和z互质就行，实际使用的RSA算法里面e默认取的定值65537，换算成hex就是上面看到的010001。我们甚至可以在用openssl genrsa生成私钥的时候用开启-3这个选项来指定e值为3 。

3）找一个自然数d，使得(e * d) mod n = 1 ，这个符合条件的d就是private exponent。

于是计算所得的(n, e)就是公钥，对应的(n, d)就是私钥。所以无论是私钥还是公钥，它其实并不是一个数，而是一对数。假设明文为x，那么加密就是计算x的e次幂对n取模；记得到的密文为y，解密就是计算y的d次幂对n取模。

**能否由RSA私钥导出对应公钥**

从上面asn1parse解析私钥的结果可以看出，私钥文件是包含对应的公钥信息的，n、e和d这三个重要整数都直接被记录，所以openssl工具可以直接由私钥导出公钥：

```
> openssl rsa -in key.encrypted -out pubkey -passin pass:12345 -pubout
> cat pubkey
-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
```

**私钥能用其它格式存储吗**

上面例子中的RSA私钥是用PKCS#1格式保存的，除此之外的常用格式还有PKCS#8，它可以简单理解成PKCS#1的升级版。PKCS#1只能用来保存RSA算法生成的密钥，而PKCS#8规范可以支持其它算法所生成的密钥。openssl pkcs8工具可以完成两种格式的转换。

```
> openssl pkcs8 -in key.encrypted -out key.encrypted.pkcs8 -passin pass:12345 -passout pass:12345 -topk8
> cat key.encrypted.pkcs8
-----BEGIN ENCRYPTED PRIVATE KEY-----
...
-----END ENCRYPTED PRIVATE KEY-----
```

可以看到，PKCS#8格式的私钥即使被加密过，文件中也没有出现PKCS#1中Proc-Type和DEK-Info这两行header，只是文件头尾多了ENCRYPTED字样。

再拿asn1parse解析一下内容：

```
> openssl asn1parse -in key.encrypted.pkcs8
    0:d=0  hl=4 l=1257 cons: SEQUENCE
    4:d=1  hl=2 l=  27 cons: SEQUENCE
    6:d=2  hl=2 l=   9 prim: OBJECT            :pbeWithMD5AndDES-CBC
   17:d=2  hl=2 l=  14 cons: SEQUENCE
   19:d=3  hl=2 l=   8 prim: OCTET STRING      [HEX DUMP]:4E1AF1084A2B8985
   29:d=3  hl=2 l=   2 prim: INTEGER           :0800
   33:d=1  hl=4 l=1224 prim: OCTET STRING      [HEX DUMP]:...
```

PKCS#8中是直接将算法ID存在ASN.1中的，而PKCS#1是通过-----BEGIN RSA PRIVATE KEY-----来说明该文件是一个RSA密钥的。于是在用私钥去解密数据的时候，需要先解析PKCS#8文件获得对应的加密算法。可以想象，如果将来需要为新加密算法提供支持，为它分配一个新ID即可，私钥文件格式本身保持不变，所以PKCS#8比PKCS#1格式更加灵活。

**ssh-keygen生成的公钥格式为什么和OpenSSL生成的不大一样**

ssh-keygen生成的公钥是默认的OpenSSH格式，这是一个私有格式。

```
> cat id_rsa.pub
ssh-rsa [...] user@MacBookPro
```

内容被空格分成三段，从左到右依次是算法名称，公钥以及备注。公钥本身是按照长度加数据的方式编码的，长度用一个32位定长的整型表示，数据是变长。长度和数据都是以对应的hex string来保存的，最后整体用base64编码。

```
> cat id_rsa.pub | cut -d ' ' -f2 | base64 -d | hexdump
0000000 00 00 00 07 73 73 68 2d 72 73 61 00 00 00 03 01
0000010 00 01 00 00 02 01 00 c8 70 7e b9 40 62 3c df c2
0000020 4e f9 02 e1 53 9f f3 aa 7d 6f fb 62 ec 35 dc 4f
...
```

00 00 00 07是后面紧接的字符串“ssh-rsa”长度，73 73 68 2d 72 73 61是“ssh-rsa”的ASCII值，没错，“ssh-rsa”又被重复编码了一次；00 00 00 03是下一段整数的长度，01 00 01是不是很熟悉？对，它就是e值65537的hex编码，其余部分就是模数n的长度和值了。上面的结果用肉眼大致能看，这里我用python写了一个简单的[解析器](https://github.com/flyingice/shell-utility/blob/master/openssh_pubkey_parser.py)可以更直观地呈现结果：

```
> python openssh_pubkey_parser.py -i id_rsa.pub
Algorithm: ssh-rsa
Exponent: 65537
Modulus: length=513 bytes
00 c8 70 7e b9 40 62 3c df c2 4e f9 02 e1 53 9f
f3 aa 7d 6f fb 62 ec 35 dc 4f 8e fd 62 3b aa b9
...
```

这本应该是一个4096位的公钥，也就是512个字节，但是结果显示模数n的长度是513个字节。原因是最前面用00填充了一个字节，目的是为了避免n被解析成一个负整数。

**如何将OpenSSH格式的公钥转换为PKCS格式**

ssh-keygen直接支持这种转换，运行命令：

```
ssh-keygen -e -f id_rsa.pub -m pem > newkey.pub
```