---
layout: post
title: Reading Notes - Prudent Engineering Practice for Cryptographic Protocols
date: 2015-07-10 22:06:59
tags: security
---

##1

最近在做一个关于支付的小需求，其中有个关于支付密码的地方，由于运维短时间内提供不了HTTPS，所以需要设计一个简单的加密协议，保证密码不能泄露，不能被重放攻击。

如何做？

使用RSA算法，客户端内置一个公钥（或者动态获取），服务器保存一个私钥，

```plain
1. server -------------nonce-----------> client
2. server <---encrypt(passowrd+nonce)--- client

server decrypt(message) and check(password+nonce)
```

当然密码存储不当也存在泄露的问题，解决这个问题可以使用PBKDF2([rfc2898](https://tools.ietf.org/html/rfc2898))。至于迭代次数可以参考stackexchange中的这个[问题](http://security.stackexchange.com/questions/3959/recommended-of-iterations-when-using-pkbdf2-sha256)。

另外终端安全坑太大，暂时不考虑。

然后又想起读书的时候上《安全协议》那门课读的那些论文，其中有一篇感觉相当经典，所以又找出来读了一遍。

##2

[Prudent Engineering Practice for Cryptographic Protocols](http://ieeexplore.ieee.org/xpl/articleDetails.jsp?arnumber=481513&filter%3DAND%28p_IS_Number%3A10294%29)

### Principle 1
>Every message should say what it means: the interpretation of the message should depend only on its content. It should be possible to write down a straightforward English sentence describing the content -- though if there is a suitable formalism available that is good too.

### Principle 2>The conditions for a message to be acted upon should be clearly set out so that someone reviewing a design may see whether they are acceptable or not.

### Principle 3
>If the identity of a principal is essential to the meaning of a message, it is prudent to mention the principal’s name explicitly in the message.

### Principle 4>Be clear as to why encryption is being done. Encryption is not wholly cheap, and not asking precisely why it is being done can lead to redundancy. Encryption is not synonymous with security, and its improper use can lead to errors.
### Principle 5>When a principal signs material that has already been encrypted, it should not be inferred that the principal knows the content of the message. On the other hand, it is proper to infer that the principal that signs a message and then encrypts it for privacy knows the content of the message.

### Principle 6>Be clear what properties you are assuming about nonces. What may do for ensuring temporal succession may not do for ensuring association -- and perhaps association is best established by other means.

### Principle 7>The use of a predictable quantity (such as the value of a counter) can serve in guaranteeing newness, through a challenge-response exchange. But if a predictable quantity is to be effective, it should be protected so that an intruder cannot simulate a challenge and later replay a response.

### Principle 8
>If timestamps are used as freshness guarantees by reference to absolute time, then the difference between local clocks at various machines must be much less than the allowable age of a message deemed to be valid. Furthermore, the time maintenance mechanism everywhere becomes part of the trusted computing base.

### Principle 9>A key may have been used recently, for example to encrypt a nonce, yet be quite old, and possibly compromised. Recent use does not make the key look any better than it would otherwise.

### Principle 10>If an encoding is used to present the meaning of a message, then it should be possible to tell which encoding is being used. In the common case where the encoding is protocol dependent, it should be possible to deduce that the message belongs to this protocol, and in fact to a particular run of the protocol, and to know its number in the protocol.

### Principle 11>The protocol designer should know which trust relations his protocol depends on, and why the dependence is necessary. The reasons for particular trust relations being acceptable should be explicit though they will be founded on judgement and policy rather than on logic.
