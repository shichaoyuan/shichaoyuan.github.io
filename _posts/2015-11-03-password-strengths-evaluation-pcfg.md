---
title: Password Strengths Evaluation 2 (PCFG)
date: 2015-11-03 22:35:12
layout: post
tags:
 - security
 - password
mathjax: true
---

## 1

上一篇谈了[zxcvbn](https://github.com/dropbox/zxcvbn)这个密码强度评估器：

1. 预先设定密码的模式，比如l33t、键盘连续字符、连续重复字符、日期、字典等；
2. 然后让待测密码匹配可能的模式；
3. 计算entropy最小的模式串，最后估算破解时间。

这里有几个问题：

1. 为什么要预先设定这些模式？而不是其他的模式？
2. 这些模式计算的entropy有效么？

要回答这些问题，我们先回到zxcvbn针对的攻击者的角度：

1. 构建字典，设想混淆密码的模式；
2. 基于字典根据预先设定的模式猜测密码。

可见如果攻防双方构建的字典、设定的模式差不多的话，那么这种评估方法是有效的，而且现实情况中这个假设基本上是成立。谈到密码破解，能想到的可能只有[John the Ripper](http://www.openwall.com/john/)。

在更短的时间内破解出更多的密码，无论在利益上还是荣誉上都是有巨大的驱动的，所以有些聪明的脑袋想出了另外一条思路——机器学习。比较有代表性的是Matt Weir, Sudhir Aggarwal, Breno de Medeiros, Bill Glodek在2009年提出的[《Password Cracking Using Probabilistic Context-Free Grammars》](http://ieeexplore.ieee.org/xpl/articleDetails.jsp?arnumber=5207658)，文章中给出的实验结果显示PCFG可以比John the Ripper多破解28%到129%的密码。

面对新型的攻击方式，防御的方法也得进行改变，基于PCFG如何做密码强度评估呢？Shiva Houshmand和Sudhir Aggarwal于2012年发表的[《Building Better Passwords using Probabilistic Techniques》](http://dl.acm.org/citation.cfm?doid=2420950.2420966)给出了一个方案。

## 2

什么是PCFG？

其中的Context-Free Grammar应该都很熟悉，[JLS](http://docs.oracle.com/javase/specs/jls/se8/html/index.html)就是用此描述程序的语法。可以将其理解为一系列的匹配规则，从左边匹配右边，最终匹配到终止字符，并且每次匹配不依赖上一次。加上Probabilistic，实际上就是为每次规则匹配赋予一个概率，比如：

```
IntegerLiteral:
  DecimalIntegerLiteral   0.2
  HexIntegerLiteral       0.3
  OctalIntegerLiteral     0.1
  BinaryIntegerLiteral    0.4
```

因为每次匹配是独立的，所以可以将匹配过程的概率值乘起来，如果规则附加的概率表示出现频率，那么最终的概率可以估算该语句出现频率。

PCFG传统的应用领域是自然语言处理，用于选择最合适的解析语法树。如果将密码理解为一种语言，自然而然就可以将PCFG迁移过来。通过泄露的密码库，训练PCFG模型，有了这个模型，就可以按照概率高低生成攻击的密码，同样的也可以用这个概率值表示密码强度。

## 3

如何定义密码的“语言”结构呢？

首先将字符串划分为三类：

|Data Type|Symbols|Examples|
---+---+-----
|Alpha String|abcdefghijklmnopqrstuvwxyzäö|cat|
|Digit String|0123456789|432|
|Special String|!@#%^&*()-_=+[]{};’:”,./<>? |!!|

这里用**L**表示Alpha，**D**表示Digit，**S**表示Special，那么就可以这样描述语法规则：

```
S:
  D1L3S2D1      0.75
  L3D1S1        0.25

D1:
  4             0.6
  5             0.2
  6             0.2

S1:
  !             1.0

S2:
  %%            0.9
  ^$            0.1

L3:           0.2
```

其中L3D1S1称为base structure，也就是两个Alpha、一个Digit、一个Special，对于Digit和Special直接匹配字符串，而Alpha比较特殊，只需要匹配长度。

如何使用PCFG模型计算密码的概率呢？

举个例子，`abc6!`按照上述规则计算概率为`0.25*0.2*0.2*1.0=0.01`。

那么如何训练这个模型呢？

已有大量的泄露密码数据，首先解析base structure，统计每种structure出现的频率做为概率；对于Digit和Special，针对不同的长度分别统计出现频率做为概率；而Alpha长度的规则比较特殊，通常来说训练集的数据没有代表性，这里可以通过统计字典数据计算这个概率，常用的字典可以从[Word lists...](http://www.outpost9.com/files/WordLists.html)这里找。

对于Alpha这里可以进一步优化，将大小写统计进去：

```
L3:           0.2
  ULL           0.3
  LLL           0.6
  LUL           0.1
```

**U**表示大写，**L**表示小写，这个概率可以从密码数据中统计出来。

对于攻击者来说，模型训练到此已经完成了，但是对于密码强度评估，还差一步。设想一下，一个待评估的新密码，当前的模型很可能没有覆盖其对应的结构，那么此时如何评估呢？所以我们需要对训练模型进行[probability smoothing](http://doc.utwente.nl/85240/1/smoothing.pdf)，同时也解决了模型过拟合的问题，在我们的实现中使用了Laplace smoothing。

## 4

如何理解最后算出的概率呢？如果可以转换成上一篇提到guessing entropy就更直观了。

James L. Massey在[《Guessing and Entropy》](http://ieeexplore.ieee.org/xpl/abstractAuthors.jsp?arnumber=394764)中提出了一个由Shannon entropy推导guessing entropy下限的公式：

$$ G(X) \geq \frac{2^{H(X)}}{4}+1  $$

那么如果计算Shannon entropy呢？

$$
H(G) = H(B,R) = H(B) + H(B|R) = H(B) + \sum\nolimits_{i}[H(X_{i_1})+H(X_{i_2})+...+H(X_{i_k})]
$$

上述公式中B表示base structure，R表示各种匹配规则。

## 5

如何设定密码强度的阈值呢？

根据破解时间设定阈值，比如小于12小时为弱，大于15天为强，估算破解时间有两个思路：

1. 由guessing entropy估算
  * 具体方法与zxcvbn类似。
2. 用guessing number估算
  * 具体来说就是使用当前模型生成按照概率高低排序的猜测密码，每个密码累加上一个计算bcrypt/scrypt/PBKDF2的时间，然后就可以拟合出一条概率与耗时的曲线。

## 6

将密码理解为自然语言可以说为密码强度评估（密码破解）领域打开了一扇窗，而且窗外有各种成熟的工具可以使用，比如说Markov模型。

还有哪些玩法呢？我们下一篇再谈。
