---
layout:     post   				    # 使用的布局（不需要改）
title:      cryptopp的编译使用 				# 标题 
date:       2019-12-05 				# 时间
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - cryptopp
    - 编译
    - 加密
---

---

在产品版本release之后，考虑到后续的使用以及资费问题，因此，需要对算法进行加密，所以考虑使用cryptopp，官网地址：<https://www.cryptopp.com/>。这篇文章记录下在这个过程中遇到的一些问题

---

## 编译安装

参考地址，很简单：<https://www.cryptopp.com/wiki/Linux#Build_and_Install_the_Library>

## 使用注意事项

### 如何使用string来记录公钥和私钥

- 使用以下代码来生成公钥和私钥，并且存储在string中：

```CPP
//此段代码是在网上搜索到的，没有记录出处，在此处感谢作者
struct KeyPairHex {
  std::string publicKey;
  std::string privateKey;
};

inline KeyPairHex RsaGenerateHexKeyPair(unsigned int aKeySize) {
  KeyPairHex keyPair;

  // PGP Random Pool-like generator
  CryptoPP::AutoSeededRandomPool rng;

  // generate keys
  CryptoPP::RSA::PrivateKey privateKey;
  privateKey.GenerateRandomWithKeySize(rng, aKeySize);
  CryptoPP::RSA::PublicKey publicKey(privateKey);

  // save keys
  publicKey.Save( CryptoPP::HexEncoder(
                    new CryptoPP::StringSink(keyPair.publicKey)).Ref());
  privateKey.Save(CryptoPP::HexEncoder(
                    new CryptoPP::StringSink(keyPair.privateKey)).Ref());
  return keyPair;
}
```

- 如何使用存储在string中的钥匙

```CPP

  CryptoPP::RSA::PrivateKey privateKey;
  privateKey.Load(CryptoPP::StringSource(aPrivateKeyStrHex, true,
                                         new CryptoPP::HexDecoder()).Ref());

```

### RSA

关于RSA不做过多介绍，不太懂的可以去参看维基百科。

关于RSA的加密解密和签名验签参考<https://www.cnblogs.com/pcheng/p/9629621.html>

- RSA加密

```CPP
inline std::string RSAEncryptString(const std::string &aPublicKeyStrHex,
                                    const std::string &aMessage, const std::string &aSeed)
{
    // decode and load public key (using pipeline)
    CryptoPP::RSA::PublicKey publicKey;
    publicKey.Load(CryptoPP::StringSource(aPublicKeyStrHex, true,
                                          new CryptoPP::HexDecoder())
                       .Ref());
    CryptoPP::RSAES_OAEP_SHA_Encryptor pub(publicKey);
    CryptoPP::RandomPool randPool;
    randPool.IncorporateEntropy((CryptoPP::byte *)aSeed.c_str(), aSeed.length());

    std::string result;
    CryptoPP::StringSource(aMessage, true, new CryptoPP::PK_EncryptorFilter(randPool, pub, new CryptoPP::HexEncoder(new CryptoPP::StringSink(result))));
    return result;
}
```

- RSA解密

```CPP
inline std::string RSADecryptString(const std::string &strPriv, const std::string &ciphertext)
{
    // decode and load private key (using pipeline)
    CryptoPP::RSA::PrivateKey privateKey;
    privateKey.Load(CryptoPP::StringSource(strPriv, true,
                                           new CryptoPP::HexDecoder())
                        .Ref());
    CryptoPP::RSAES_OAEP_SHA_Decryptor priv(privateKey);

    std::string result;
    CryptoPP::StringSource(ciphertext, true, new CryptoPP::HexDecoder(new CryptoPP::PK_DecryptorFilter(GlobalRNG(), priv, new CryptoPP::StringSink(result))));
    return result;
}
```

- RSA签名

```CPP
inline std::string RsaSignString(const std::string &aPrivateKeyStrHex,
                                 const std::string &aMessage)
{

    // decode and load private key (using pipeline)
    CryptoPP::RSA::PrivateKey privateKey;
    privateKey.Load(CryptoPP::StringSource(aPrivateKeyStrHex, true,
                                           new CryptoPP::HexDecoder())
                        .Ref());

    // sign message
    std::string signature;
    Signer signer(privateKey);
    CryptoPP::AutoSeededRandomPool rng;

    CryptoPP::StringSource ss(aMessage, true,
                              new CryptoPP::SignerFilter(rng, signer,
                                                         new CryptoPP::HexEncoder(
                                                             new CryptoPP::StringSink(signature))));

    return signature;
}
```

- RSA验签

```CPP
inline bool RsaVerifyString(const std::string &aPublicKeyStrHex,
                            const std::string &aMessage,
                            const std::string &aSignatureStrHex)
{

    // decode and load public key (using pipeline)
    CryptoPP::RSA::PublicKey publicKey;
    publicKey.Load(CryptoPP::StringSource(aPublicKeyStrHex, true,
                                          new CryptoPP::HexDecoder())
                       .Ref());

    // decode signature
    std::string decodedSignature;
    CryptoPP::StringSource ss(aSignatureStrHex, true,
                              new CryptoPP::HexDecoder(
                                  new CryptoPP::StringSink(decodedSignature)));

    // verify message
    bool result = false;
    Verifier verifier(publicKey);
    CryptoPP::StringSource ss2(decodedSignature + aMessage, true,
                               new CryptoPP::SignatureVerificationFilter(verifier,
                                                                         new CryptoPP::ArraySink((CryptoPP::byte *)&result,
                                                                                                 sizeof(result))));

    return result;
}

```

### AES加密

- AES密匙

```CPP
CryptoPP::AutoSeededRandomPool prng;
CryptoPP::SecByteBlock key(CryptoPP::AES::DEFAULT_KEYLENGTH);
key[0] = 0x00;key[1] = 0x00;key[2] = 0x00; key[3] = 0x00;
key[4] = 0x00;key[5] = 0x00;key[6] = 0x00; key[7] = 0x00;
key[8] = 0x00;key[9] = 0x00;key[10] = 0x00; key[11] = 0x00;
key[12] = 0x00;key[13] = 0x00;key[14] = 0x00; key[15] = 0x00;

CryptoPP::byte iv[ CryptoPP::AES::BLOCKSIZE ];
iv[0] = 0x00;iv[1] = 0x00;iv[2] = 0x00; iv[3] = 0x00;
iv[4] = 0x00;iv[5] = 0x00;iv[6] = 0x00; iv[7] = 0x00;
iv[8] = 0x00;iv[9] = 0x00;iv[10] = 0x00; iv[11] = 0x00;
iv[12] = 0x00;iv[13] = 0x00;iv[14] = 0x00; iv[15] = 0x00;

```

- AES加密

```CPP
CryptoPP::CBC_Mode<CryptoPP::AES >::Encryption aes_encry;
aes_encry.SetKeyWithIV( key, key.size(), iv );
string aescipher;
CryptoPP::StringSource aese(string,true,new CryptoPP::StreamTransformationFilter(aes_encry,new CryptoPP::StringSink(aescipher)));
```

- AES解密

```CPP
CryptoPP::CBC_Mode< CryptoPP::AES >::Decryption d;
d.SetKeyWithIV( key, key.size(), iv );
string aescipher;
CryptoPP::StringSource aesd(string, true, new CryptoPP::StreamTransformationFilter(d,new CryptoPP::StringSink( aescipher)));
```

## 使用中遇到的问题

单独使用demo时，没有任何问题；然而在跟libtorch一起使用的时候，就出现了问题，一直提示找不到符号，查了几遍都没有问题，不过使用nm查看的时候，立刻发现了问题：

- 编译出的cryptopp.so中的符号是：std::__cxx11::basic_string

- 程序提示需要的符号是：std::string

参考文档: <https://blog.csdn.net/soipray/article/details/52693444>

有了解决方法，根据官网的说明，在编译的时候设置CXXFLAGS，结果发现没有用，出来的还是一样。因此直接在gnumakefile里面手动改，
> CXXFLAGS += -D_GLIBCXX_USE_CXX11_ABI=0

重新编译通过
