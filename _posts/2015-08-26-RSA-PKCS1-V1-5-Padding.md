---
layout: post
title: BigInteger#toByteArray引发的处理PKCS1Padding的问题
date: 2015-08-26 08:30:46
tags:
- security
- RSA
---

##1

之前的[文章](http://shichaoyuan.github.io/2015/07/10/notes-pepfcp/)提到过使用RSA加密传输密码的应用场景，在和H5联调的时候遇到了不少问题……

```java
Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
cipher.init(Cipher.DECRYPT_MODE, prikey);
```

后端限定的RSA解密实现如上所示；H5前端发来的消息，偶尔会报下述异常：

```java
Caused by: javax.crypto.BadPaddingException: Data must start with zero
	at sun.security.rsa.RSAPadding.unpadV15(RSAPadding.java:308)
	at sun.security.rsa.RSAPadding.unpad(RSAPadding.java:255)
	at com.sun.crypto.provider.RSACipher.a(DashoA13*..)
	at com.sun.crypto.provider.RSACipher.engineDoFinal(DashoA13*..)
	at javax.crypto.Cipher.doFinal(DashoA13*..)
```

之前概率不大，一直没太关心，最近发现失败的情况还挺多，所以查了一下，顺便做个笔记。

##2

首先跟一下异常抛出的位置，线上用的是jdk1.6，所以先到[http://hg.openjdk.java.net/jdk6/jdk6/jdk/](http://hg.openjdk.java.net/jdk6/jdk6/jdk/)这里下载一下源码，定位到文件，看了一下，实现上差别挺大。

怎么办？使用Bytecode反编译吧。

抛异常的位置在这儿：

```java
    private byte[] unpadV15(final byte[] array) throws BadPaddingException {
        int n = 0;
        if (array[n++] != 0) {
            throw new BadPaddingException("Data must start with zero");
        }
        if (array[n++] != this.type) {
            throw new BadPaddingException("Blocktype mismatch: " + array[1]);
        }
        while (true) {
            final int n2 = array[n++] & 0xFF;
            if (n2 == 0) {
                final int n3 = array.length - n;
                if (n3 > this.maxDataSize) {
                    throw new BadPaddingException("Padding string too short");
                }
                final byte[] array2 = new byte[n3];
                System.arraycopy(array, array.length - n3, array2, 0, n3);
                return array2;
            }
            else {
                if (n == array.length) {
                    throw new BadPaddingException("Padding string not terminated");
                }
                if (this.type == 1 && n2 != 255) {
                    throw new BadPaddingException("Padding byte not 0xff: " + n2);
                }
                continue;
            }
        }
    }
```

也就是说解密之后开始的第一个byte不是0……

好的，我们再看一下异常的消息：

```java
byte[] passwordBytes = new BigInteger(encryptedPassword, 16).toByteArray()
```

后端这里使用上述方式，将十六进制的消息转成byte数组，打印出来：


```plain
异常的消息：
93e15ed1e75e5e5f51d45498fd704d96a0e7fc6892bca10dc2976bb1d39659e0ea3eb0b586dcb2f5f491e8d3f4db07e5856406634f64ac0c307181a9a52d4fb75dc872b3bb070ea6bacdcaa0a78cdd9330dc91d33141852669068896754161dee37a51fcb1d74690ab2e5387cede616adf29472058f4d669500710b68d0d8eeea0fb2fc4a703e13ef3179dba8ba00c6a59ab9048c8847af06c2d9e82a00355bae44762fda6ceee6ac69c4c47638a17ba39a4135c6ee0175f7c8e0c46f280925a1f15e5f9ea43bfa64d6c16ea59f011605c32b7f4a0c5b09dd0c178b68e0cce3413ff03db13eddc4bc3a84062310d3f540d34bca4bfc648699a83145b21753d6f

  0: 00000000 10010011 11100001 01011110 11010001 11100111 01011110 01011110
  8: 01011111 01010001 11010100 01010100 10011000 11111101 01110000 01001101
……省略  
248: 01101001 10011010 10000011 00010100 01011011 00100001 01110101 00111101
256: 01101111


正常的消息：
5eb2dc0d2c675a60830ea92ccb3c514a26fcdc6a7d2621d905ba0cb3eb02ce0bac052cf1833022a0a9ebccf546b0a3a8d50f165a6ae0b7c86840f09b99e32bf25e614ff2ca59bdc220377effde60f6d464d13f8403ad0f5e03fa79985eb0c842743eb3179211a0438d7dabe8567c11871b5d1182d1cb8cd58b9a06777463c62f79d0c1fdfb01a0a637906919e58ca9cd9d8d2e883655e5b7401ce16e0d507bea97f8a54a44231b88b4a7d65133949aacdde8c0e672a21f6bc8277ed0abc8ac4239a909007db3b8063c82d70af7df665550cb79fded779e9f92ff6c4c6b2cda17879b6a3e854b77224ebec5b0095b2f1123e7730d62c188eac9f6019dd7cff3a3

  0: 01011110 10110010 11011100 00001101 00101100 01100111 01011010 01100000
  8: 10000011 00001110 10101001 00101100 11001011 00111100 01010001 01001010
……省略
240: 00100011 11100111 01110011 00001101 01100010 11000001 10001000 11101010
248: 11001001 11110110 00000001 10011101 11010111 11001111 11110011 10100011
```

我们发现异常的消息前面多了一个为0的byte，问题出在BigInteger的toByteArray方法：

```java
    public byte[] toByteArray() {
        final int n = this.bitLength() / 8 + 1;
        final byte[] array = new byte[n];
        int i = n - 1;
        int n2 = 4;
        int int1 = 0;
        int n3 = 0;
        while (i >= 0) {
            if (n2 == 4) {
                int1 = this.getInt(n3++);
                n2 = 1;
            }
            else {
                int1 >>>= 8;
                ++n2;
            }
            array[i] = (byte)int1;
            --i;
        }
        return array;
    }
```

如果bit长度是8的倍数，前面就会多一个为0的byte……所以自己写一个转换方法：

```java
public static byte[] hexStringToByteArray(String s) {
    int len = s.length();
    byte[] data = new byte[len / 2];
    for (int i = 0; i < len; i += 2) {
        data[i / 2] = (byte) ((Character.digit(s.charAt(i), 16) << 4)
                                 + Character.digit(s.charAt(i+1), 16));
    }
    return data;
}
```

测试了几次都没有问题……

所以偶尔得多看看JDK的实现，一不小心使用的姿势就不对了……

##3

最后贴点儿资料

[rfc3447](https://tools.ietf.org/html/rfc3447) RSA加密规范

[OpenSSL 1024 bit RSA Private Key Breakdown](http://etherhack.co.uk/asymmetric/docs/rsa_key_breakdown.html) Key文件描述



