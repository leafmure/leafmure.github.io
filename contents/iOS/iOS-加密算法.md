---
title: iOS 加密算法
date: 2021-09-13 20:02:08
categories:
- iOS
tags:
- 加密算法
keywords: AES,DES,MD5
description:
images:
---

数据加密的基本过程就是对原来为明文的文件或数据按某种算法进行处理，通常用于达到以下目的：

- 保密性：防止内容被第三方获取
- 数据完整性：防止数据被篡改
- 身份验证：确保数据是正确方发送的
<!-- more -->
### 摘要算法
消息摘要算法的主要特征是加密过程不需要密钥，并且经过加密的数据无法被解密，目前可以被解密逆向的只有CRC32算法，只有输入相同的明文数据经过相同的消息摘要算法才能得到相同的密文。保证信息的完整性和不可否认性的方法。数据的完整性是指信宿接收到的消息一定是信源发送的信息，而中间绝无任何更改.

消息摘要算法主要应用在“数字签名”领域，作为对明文的摘要算法。著名的摘要算法有RSA公司的MD5算法和SHA-1算法及其大量的变体。

#### MD5
MD是应用非常广泛的一个算法家族，尤其是 MD5（Message-Digest Algorithm 5，消息摘要算法版本5），它由MD2、MD3、MD4发展而来，由Ron Rivest（RSA公司）在1992年提出，目前被广泛应用于数据完整性校验、数据（消息）摘要、数据加密等。MD2、MD4、MD5 都产生16字节（128位）的校验值，一般用32位十六进制数表示。MD2的算法较慢但相对安全，MD4速度很快，但安全性下降，MD5比MD4更安全、速度更快。

特点：
- 不可逆性：字符串的转换过程是不可逆的,不能通过加密结果,反向推导出原始内容
- 压缩性：任意长度的数据，算出的MD5值长度都是固定的，把一个任意长度的字节串变换成一定长度的十六进制的大整数。
- 易计算性：从原数据计算出MD5数据很容易
- 抗修改性：对原数据进行任何改动，那怕只修改一个字节，所得到的MD5值都有很大区别
- 弱抗碰撞：已知原数据和其MD5 值,想找到一个具有相同 MD5 值的数据(即伪造数据)是非常困难的
- 强抗碰撞：想找到两个不同数据,使他们具有相同的 MD5 值,是非常困难的

#### MD5 算法原理：
##### 1. 数据填充
对数据进行填充，使数据的长度对512取模得448，设数据长度为X，即满足：X mod 512 = 448，根据此公式得出需要填充的数据长度，在数据的后面填充一个1和无数个0，直到满足上面的条件时才停止用 0 对数据的填充，将数据长度扩充至：N * 512 + 448。

##### 2. 添加数据长度
在第一步结果之后附加一个以64位二进制表示的填充前数据长度（单位为Bit），如果二进制表示的填充前数据长度超过64位，则取低64位。在此步骤进行完毕后，数据的长度为：N * 512 + 448 + 64 = (N+1）* 512，最终数据长度就是512的整数倍。

##### 3. 数据处理
MD5运算要用到一个128位的MD5缓存器，用来保存中间变量和最终结果。该缓存器又可看成是4个32位的寄存器A、B、C、D，初始化为：
```Objective-C
A = 0x67452301
B = 0x0EFCDAB89
C = 0x98BADCFE
D = 0x10325476
```

以512位分组来处理数据，且每一分组又被划分为16个32位子分组，每组数据进行4轮的逻辑处理，最后的输出由四个32位分组组成，将这四个32位分组级联后将生成一个128位散列值。

对每个数据段都要进行4轮的逻辑处理，在4轮中分别使用4个不同的函数F、G、H、I:
```Objective-C
F(X,Y,Z) = (X & Y) | ((~X) & Z);
G(X,Y,Z) = (X & Z) | (Y & (~Z)); 
H(X,Y,Z) = X ^ Y ^ Z; 
I(X,Y,Z) = Y ^ (X | (~Z));
```


MD5 简单实现：
```Objective-C
导入CommonDigest.h：
#import <CommonCrypto/CommonDigest.h>
  
封装MD5签名分类方法
- (NSString *)md5String{
   CC_MD5_CTX md5;
   CC_MD5_Init (&md5);
    CC_MD5_Update (&md5, [self UTF8String], (CC_LONG)[self length]);
   unsigned char digest[CC_MD5_DIGEST_LENGTH];
   CC_MD5_Final (digest, &md5);
   NSString *mdString = [NSString stringWithFormat:   @"%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x", 
   digest[0], digest[1],
   digest[2],  digest[3],
   digest[4],  digest[5],
   digest[6],  digest[7],
   digest[8],  digest[9],
   digest[10], digest[11],
   digest[12], digest[13],
   digest[14], digest[15]];
   
   return mdString;
}

- (NSData *)MD5Sum  {
    unsigned char hash[CC_MD5_DIGEST_LENGTH];
    (void) CC_MD5( [self bytes], (CC_LONG)[self length], hash );
    return ( [NSData dataWithBytes: hash length: CC_MD5_DIGEST_LENGTH] );
}

使用：

NSString => MD5String
NSSting *md5String = [@"字符串" md5String];
# 16 位md5
NSSting *md5String2 = [md5String substringWithRange:NSMakeRange(8, 16)];

NSData => MD5Data 
NSData *data = [data  MD5Sum];
```
#### SHA (Secure Hash Algorithm 安全散列算法)
SHA（Secure Hash Algorithm）是由美国专门制定密码算法的标准机构——美国国家标准技术研究院（NIST）制定的，SHA系列算法的摘要长度分别为：SHA-1为20字节（160位）、SHA-224为32字节（224位）、SHA-256为32字节（256位）、SHA-384为48字节（384位）、SHA-512为64字节（512位），由于它产生的数据摘要的长度更长，因此更难以发生碰撞，因此较之MD5更为安全，它是未来数据摘要算法的发展方向。由于SHA系列算法的数据摘要长度较长，因此其运算速度与MD5相比，也相对较慢。

目前SHA1的应用较为广泛，主要应用于CA和数字证书中，另外在目前互联网中流行的BT软件中，也是使用SHA1来进行文件校验的。

SHA 简单实现：
```Objective-C
NSString *text = @"sdadasda";
const char *cstr = [text cStringUsingEncoding:NSUTF8StringEncoding];
NSData *data = [NSData dataWithBytes:cstr length:text.length];
//使用对应的CC_SHA1,CC_SHA256,CC_SHA384,CC_SHA512的长度分别是20,32,48,64
uint8_t digest[CC_SHA1_DIGEST_LENGTH];
//使用对应的CC_SHA256,CC_SHA384,CC_SHA512
unsigned int length = (int)data.length;
CC_SHA1(data.bytes, length, digest);

NSMutableString* output = [NSMutableString stringWithCapacity:CC_SHA1_DIGEST_LENGTH * 2];
for(int i = 0; i < CC_SHA1_DIGEST_LENGTH; i++)
    [output appendFormat:@"%02x", digest[i]];
```

### 对称加密
对称加密指的是加密和解密都使用用同一密钥
- 优点：算法公开、计算量小、加密速度快、加密效率高。
- 缺点：加密、解密用同一密钥，对于密钥保存要有很高的安全性

#### DES
> DES全称为Data Encryption Standard，即数据加密标准，是一种使用密钥加密的块算法，1977年被美国联邦政府的国家标准局确定为联邦资料处理标准（FIPS），并授权在非密级政府通信中使用，随后该算法在国际上广泛流传开来。需要注意的是，在某些文献中，作为算法的DES称为数据加密算法（Data Encryption Algorithm,DEA），已与作为标准的DES区分开来。

DES设计中使用了分组密码设计的两个原则：混淆（confusion）和扩散(diffusion)，其目的是抗击敌手对密码系统的统计分析。混淆是使密文的统计特性与密钥的取值之间的关系尽可能复杂化，以使密钥和明文以及密文之间的依赖性对密码分析者来说是无法利用的。扩散的作用就是将每一位明文的影响尽可能迅速地作用到较多的输出密文位中，以便在大量的密文中消除明文的统计结构，并且使每一位密钥的影响尽可能迅速地扩展到较多的密文位中，以防对密钥进行逐段破译。

DES算法使用的密钥是64位，实际用到了56位，第8、16、24、32、40、48、56、64位是校验位。


#### 3DES
3DES（即Triple DES）是DES向AES过渡的加密算法，它使用3条56位的密钥对数据进行三次加密。是DES的一个更安全的变形。它以DES为基本模块，通过组合分组方法设计出分组加密算法。比起最初的DES更为安全。

该方法使用两个密钥，执行三次DES算法，加密的过程是加密-解密-加密，解密的过程是解密-加密-解密。
```
3DES加密过程为：C=Ek3(Dk2(Ek1(P)))
3DES解密过程为：P=Dk1(EK2(Dk3(C)
```

#### AES
> 密码学中的高级加密标准（Advanced Encryption Standard，AES），又称Rijndael加密法，是美国联邦政府采用的一种区块加密标准。

严格地说，AES和Rijndael加密法并不完全一样（虽然在实际应用中二者可以互换），因为Rijndael加密法可以支持更大范围的区块和密钥长度：AES的区块长度固定为128位，密钥长度则可以是128，192或256位；而Rijndael使用的密钥和区块长度可以是32位的整数倍，以128位为下限，256位为上限。加密过程中使用的密钥是由Rijndael密钥生成方案产生。

#### RC算法
RC算法是Ron Rivest发明的一组对称密钥加密算法。“RC”可能代表Rivest密码，或者更非正式地说，代表Ron密码。尽管名称相似，但这些算法在很大程度上是不相关的。到目前为止，已有六种RC算法：
- RC1从未出版过。

- RC2是1987年开发的64位分组密码。

- RC3在使用前就被抛弃。

- RC4是世界上使用最广泛的流密码。

- RC5是1994年开发的32/64/128位分组密码。

- RC6是一种基于RC5的128位分组密码，是1997年开发的AES最终产品。

#### 对称加密的实现（CommonCrypto）
加密模式有：
- ECB：电码本模式（Electronic Codebook Book），这种模式是将整个明文分成若干段相同的小段，然后对每一小段进行加密。
- CBC：密码分组链接模式（Cipher Block Chaining）， 这种模式是先将明文切分成若干小段，然后每一小段与初始块或者上一段的密文段进行异或运算后，再与密钥进行加密。
- CTR：计算器模式（Counter），计算器模式不常见，在CTR模式中， 有一个自增的算子，这个算子用密钥加密之后的输出和明文异或的结果得到密文，相当于一次一密。这种加密方式简单快速，安全可靠，而且可以并行加密，但是在计算器不能维持很长的情况下，密钥只能使用一次。
- CFB：密码反馈模式（Cipher FeedBack）
- OFB：输出反馈模式（Output FeedBack）

不同的加密算法对于加密内容的分组大小不一致，当内容不满足长度要求就根据填充方式去填充以达到符合位数长度，填充方式有：
- ccNoPadding 不填充
- ccPKCS7Padding
- ANSIX923
- ISO10126
- PKCS7
- Zero
```
需要填充的字节长度 = (块长度 - (数据长度 % 块长度))

ANSIX923:
在填充字节序列中最后一个字节填充为需要填充的字节长度值, 填充字节中其余字节均填充数字零.

例：假定块长度为8 ，数据长度为 10,则填充字节数等于 6
数据： FF FF FF FF FF FF FF FF | FF DD
填充后： FF FF FF FF FF FF FF FF | FF DD 00 00 00 00 00 06

ISO10126:
在填充字节序列中最后一个字节填充为需要填充的字节长度值, 填充字节中其余字节均填充随机数值.

例：假定块长度为 16，数据长度为 9,则填充字节数等于 7
数据：FF FF FF FF FF FF FF FF FF
填充后：FF FF FF FF FF FF FF FF FF 75 64 C3 82 A5 26 07 

PKCS7:
在填充字节序列中所有字节填充为需要填充的字节长度值
例：假定块长度为 8，数据长度为 3,则填充字节数等于 5
数据：FF FF FF
填充后：FF FF FF 05 05 05 05 05

Zero:
在填充字节序列中所有字节填充为0x00
例：假定块长度为 8，数据长度为 3, 则填充字节数等于 5。
数据：FF FF FF
填充后：FF FF FF 00 00 00 00 00
```
在 iOS 官方提供一个加密框架 CommonCrypto，通过它可以方便的实现 DES、AES、3DES 等算法
```Objective-C
// 注意：各算法对于密钥的长度要求不一致，这里以 DES-CBC 加密 为例
NSString *key = @"1234567812345678";
NSString *iv = @"123";
NSString *plainText = @"sdadasda";

// 密钥、iv 变量
NSMutableData * keyData, * ivData;
keyData = [[key dataUsingEncoding: NSUTF8StringEncoding] mutableCopy];
ivData = [[iv dataUsingEncoding: NSUTF8StringEncoding] mutableCopy];

// 密钥、iv 变量要为8字节/64位，不足就补充至8字节/64位
[keyData setLength: 8];
[ivData setLength: 8];

NSData* plainTextData = [plainText dataUsingEncoding: NSUTF8StringEncoding];

// 创建加/解密器 CCCryptorRef
/**
 *  CCAlgorithm -> kCCAlgorithmDES 加密算法
 *  CCOperation -> kCCEncrypt(加密)、kCCDecrypt(解密)
 *  CCMode -> kCCModeCBC 加密模式
 *  CCPadding -> ccPKCS7Padding 填充方式
 *  iv -> ivData.bytes
 *  key -> keyData.bytes
 *  keyLength -> keyData.bytes.length 转Data字节数据的密钥长度
 *  CCCryptorRef -> cryptorRef 加/解密器
 */
CCCryptorRef cryptor = NULL;
CCCryptorStatus status = CCCryptorCreateWithMode(kCCEncrypt, kCCModeCBC, kCCAlgorithmDES, ccPKCS7Padding, ivData.bytes, [keyData bytes], [keyData length], NULL, 0, 0, 0, &cryptor);

// 输出数据时需要的可用空间大小
size_t bufsize = CCCryptorGetOutputLength(cryptor, (size_t)[plainTextData length], true);
// 输出已加密串的内存地址/缓冲区
void * buf = malloc( bufsize );
// 输出成功之后实际占用的空间大小.
size_t bufused = 0;
// 输出密文总大小
size_t bytesTotal = 0;
// 处理（加密、解密）一些数据，将结果将写入缓冲区 buf。
status = CCCryptorUpdate(cryptor, [plainTextData bytes], (size_t)[plainTextData length], buf, bufsize, &bufused);
if ( status != kCCSuccess ) {
    free( buf );
    return;
}

bytesTotal += bufused;

// 完成加密或解密操作，并获得最终数据输出。
status = CCCryptorFinal(cryptor, buf + bufused, bufsize - bufused, &bufused );
if ( status != kCCSuccess ) {
    free( buf );
    return;
}
   
bytesTotal += bufused;

NSData *resulltData = [NSData dataWithBytesNoCopy: buf length: bytesTotal];
NSString *resullt = [resulltData base64EncodedStringWithOptions:0];
CCCryptorRelease(cryptor);
```

### 非对称加密
非对称加密需要两个密钥：公钥 (publickey) 和私钥 (privatekey)。公钥和私钥是一对，如果用公钥对数据加密，那么只能用对应的私钥解密。如果用私钥对数据加密，只能用对应的公钥进行解密。因为加密和解密用的是不同的密钥，所以称为非对称加密。

非对称加密算法的保密性好，它消除了最终用户交换密钥的需，但是加解密速度要远远慢于对称加密。

#### RSA
> RSA是1977年由罗纳德·李维斯特（Ron Rivest）、阿迪·萨莫尔（Adi Shamir）和伦纳德·阿德曼（Leonard Adleman）一起提出的。当时他们三人都在麻省理工学院工作，RSA就是他们三人姓氏开头字母拼在一起组成的。

RSA公开密钥密码体制的原理是：根据数论，寻求两个大素数比较简单，而将它们的乘积进行因式分解却极其困难，因此可以将乘积公开作为加密密钥。

由于进行的都是大数计算，使得RSA最快的情况也比DES慢上好几倍，无论是软件还是硬件实现。速度一直是RSA的缺陷，在用于少量数据加密的情况，RSA的速度比对应同样安全级别的对称密码算法要慢1000倍左右。

##### 质数和互质关系
质数：又称素数，指在大于 1 的自然数中，除了1和自身外，不能被其他自然数整除的数，例如：2，3，5，7，11...

互质关系：对于N个自然数，他们的公因数只有1，则成N个数互质。例如：7 和 8，3 和 4，3 和 5 等。

##### 欧拉定理
对于正整数N，将小于或等于N并与N互质的数的个数记作φ(N)。
```
例如：正整数 5，小于或等于5并与5互质的数有：1，2，3，4，所以 φ(5) = 4
```
欧拉定理的引理:
- 对于质数p，则φ(p) = p-1
- 如果两个互质数a、b，则 φ(a * b) = φ(a)*φ(b)



##### 密钥生成算法
1. 随机生成两个质数 p、q，计算 n = pq，计算欧拉函数 φ(n) = (p−1)(q−1)；
2. 任意选取一个满足 e 与 φ(n) 最大公约数为 1 的整数e，即： gcd(e, φ(n)) = 1,（e > 1）
3. 确定解密密钥 d，满足 (d * e) mod φ(n) = 1，即：d * e = k * φ(n), k >= 1（k是一个整数），所以，如果知道 e 和 φ(n) ，则很容易计算出d，但根据 n 和 e 要计算出d是不可能的。
4. 公开整数n和e作为公钥，秘密保存d作为私钥

##### 加/解密算法
```
// c:密文  m:原文 (e,n):公钥  d:私钥
c = E(m) = m^e mod n

m = D(c) = c^d mod n

示例：
质数 p、q
p = 3
q = 5

公共数 n
n = p * q = 3 * 5 = 15
φ(n) = (p - 1) * (q - 1) = 2 * 4 = 8
φ(15): { 1,2,4,7,8,11,13,14} = 8

已知 φ(n) = 8，e => { 3，5，7，9，11...}，只要 e 和 φ(n) 互质就行，这里取 e = 11

私钥 (d * e) % φ(n) = (d * 11) % 8 = 1
d = 3

假设 m = 3
c = 3^11 % 15 = 177147 % 15 = 12
m = 12^3 % 15 = 1728 % 15 = 3
```

RSA的安全性依赖于大数分解，但是否等同于大数分解一直未能得到理论上的证明，也并没有从理论上证明破译。RSA的难度与大数分解难度等价。因为没有证明破解RSA就一定需要做大数分解。假设存在一种无须分解大数的算法，那它肯定可以修改成为大数分解算法，即RSA的重大缺陷是无法从理论上把握它的保密性能如何，而且密码学界多数人士倾向于因子分解不是[P类问题](https://baike.baidu.com/item/P%E7%B1%BB%E9%97%AE%E9%A2%98/22400499?fromtitle=NPC%E9%97%AE%E9%A2%98&fromid=8698778)。

目前，RSA的一些变种算法已被证明等价于大数分解。不管怎样，分解 n 是最显然的攻击方法。现在，人们已能分解140多个十进制位的大素数。因此，模数 n 必须选大些，视具体适用情况而定。

RSA算法的保密强度随其密钥的长度增加而增强。但是，密钥越长，其加解密所耗用的时间也越长。因此，要根据所保护信息的敏感程度与攻击者破解所要花费的代价值不值得以及系统所要求的反应时间来综合考虑，尤其对于商业信息领域更是如此。