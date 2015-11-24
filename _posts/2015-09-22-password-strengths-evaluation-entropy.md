---
layout: post
title: Password Strengths Evaluation 1 (Entropy)
date: 2015-09-22 20:23:32
tags:
 - security
 - password
mathjax: true
---

##1

密码应该说是最普遍的身份认证的方式，对其的攻击也有很多种，比如：暴力破解、基于字典、甚至可以通过用户信息猜测、撞库，近年来又出现了基于机器学习的方法猜测密码的黑科技。

所以就有一系列问题，哪种密码对上述攻击的抵抗性更高？如何评估密码的强度？如何引导用户选择高强度而且容易记忆的密码。

关于这个几个问题，最近也是在边学边做，所以顺便做一下笔记，这篇就谈一下目前使用最广泛的密码强度评估方法——Entropy。

##2

关于Entropy，Claude Shannon是这样解释的：

>The entropy is a statistical parameter which measures in a certain sense, how much information is produced on the average for each letter of a text in the language. If the language is translated into binary digits (0 or 1) in the most efficient way, the entropy H is the average number of binary digits required per letter of the original language.

形式化为：

$$ H(X)=-\sum\limits_{x}P(X=x)\log_{2}P(X=x) $$

将自然语言中的entropy的概念迁移到密码中来，该值表征的就是密码的不确定程度，不确定程度越高，攻击者“猜”到密码的可能性越低。

首先假设密码是随机生成的，长度固定为*l*，可选字符集大小为*b*，那么此时

$$ H=log_{2}(b^{l}) $$

entropy的值是最高的，也就是说该密码库理论上是最安全的，但是此时也就没有可用性了，因为用户根本记不住随机密码（如果没有辅助手段）。

现实中的密码库远远不是随机的，Mark Burnett曾经搜集了6百万个密码，他发现其中有大量的重复密码，出现概率最高的1000个密码占总量的91%左右，而且很多密码符合一定的模式，比如qwerty、123456，或者再进行简单的l33t替换。

##3

对于现实中的密码不能仅仅用*l*和*b*来计算entropy（当然这两个值对entropy贡献最大），那么如何基于entropy计算未知密码的强度呢？

dropbox开源了一个基于entropy的密码强度评估的类库——[zxcvbn](https://github.com/dropbox/zxcvbn)，我们看一下她是如何实现的。

首先用一下：

```shell
➜  zxcvbn git:(master) ✗ node
> var zxcvbn = require('./dist/zxcvbn.js')
undefined
> zxcvbn('Yu@n1989shiChao')
{ password: 'Yu@n1989shiChao',
  entropy: 41.911,
  match_sequence:
   [ { pattern: 'dictionary',
       i: 0,
       j: 3,
       token: 'Yu@n',
       matched_word: 'yuan',
       rank: 1237,
       dictionary_name: 'passwords',
       reversed: false,
       l33t: true,
       sub: [Object],
       sub_display: '@ -> a',
       base_entropy: 10.27262978497637,
       uppercase_entropy: 1,
       reversed_entropy: 0,
       l33t_entropy: 1,
       entropy: 12.27262978497637 },
     { pattern: 'regex',
       token: '1989',
       i: 4,
       j: 7,
       regex_name: 'recent_year',
       regex_match: [Object],
       entropy: 4.321928094887363 },
     { pattern: 'regex',
       token: 'shi',
       i: 8,
       j: 10,
       regex_name: 'alpha_lower',
       regex_match: [Object],
       entropy: 14.101319154423278 },
     { pattern: 'dictionary',
       i: 11,
       j: 14,
       token: 'Chao',
       matched_word: 'chao',
       rank: 1189,
       dictionary_name: 'passwords',
       reversed: false,
       base_entropy: 10.215532999745657,
       uppercase_entropy: 1,
       reversed_entropy: 0,
       l33t_entropy: 0,
       entropy: 11.215532999745657 } ],
  crack_time: 206805262.144,
  crack_time_display: '6 years',
  score: 4,
  calc_time: 13 }
>
```

返回结果的字段的含义：

```plain
password 密码
entropy 
match_sequence 密码匹配到的模式
crack_time 估算的破解时间，根据entropy计算
crack_time_display
score 根据crack_time映射的分值
calc_time 计算耗时
```

实际上zxcvbn累加了匹配到的模式的entropy值来评估密码强度，因为攻击者可以通过构建的字典和一些模式减小“猜”的范围。

**这种从攻击者的角度评估密码强度的思路是很可取的。**

抛开细枝末节看一下主流程：

```coffee
# 对输入的密码进行模式匹配，也就是结果中的match_sequence
matches = matching.omnimatch password
# 计算各种entropy
result = scoring.minimum_entropy_match_sequence password, matches
```

zxcvbn考虑到的模式还是挺多的：

```coffee
@dictionary_match #基于字典，也就是查看密码的子串是否出现在字典中，以及在字典中的位置
  passwords:    build_ranked_dict frequency_lists.passwords #常见的密码，概率降序
  english:      build_ranked_dict frequency_lists.english
  surnames:     build_ranked_dict frequency_lists.surnames
  male_names:   build_ranked_dict frequency_lists.male_names
  female_names: build_ranked_dict frequency_lists.female_names
@reverse_dictionary_match #翻转密码字符串
@l33t_match #l33t逆回去之后再进行dictionary_match
@spatial_match #qwerty这类键盘连续的字符串
@repeat_match #aaa这类连续的串
@sequence_match #abcd这类连续的串
@regex_match #匹配连续的符号、数字、大写、小写 （3个以上）
@date_match #匹配日期
```

得出一个匹配列表，然后按照i和j排个序，在列表中，ij区间是有重叠的，所以首先算出一个不重叠的子列表，并且要求entropy最小，这个过程可以理解为找最容易攻击的一组模式，所以就可以用这个最小的entropy做为密码强度的评估值。

```coffee
  minimum_entropy_match_sequence: (password, matches) ->
    # 计算暴力破解的基数，也就是计算entropy公式中的(b**l)
    bruteforce_cardinality = @calc_bruteforce_cardinality password # e.g. 26 for lowercase
    # 保存到位置k的最小entropy，就是个动态规划嘛
    up_to_k = []      # minimum entropy up to k.
    # for the optimal seq of matches up to k, backpointers holds the final match (match.j == k).
    # null means the sequence ends w/ a brute-force character.
    backpointers = []
    for k in [0...password.length]
      # starting scenario to try and beat:
      # adding a brute-force character to the minimum entropy sequence at k-1.
      up_to_k[k] = (up_to_k[k-1] or 0) + @lg bruteforce_cardinality
      backpointers[k] = null
      for match in matches when match.j == k
        [i, j] = [match.i, match.j]
        # see if best entropy up to i-1 + entropy of this match is less than current minimum at j.
        # 在calc_entropy中，entropy的计算方法因“匹配类型”而异
        candidate_entropy = (up_to_k[i-1] or 0) + @calc_entropy(match)
        if candidate_entropy < up_to_k[j]
          up_to_k[j] = candidate_entropy
          backpointers[j] = match

    # walk backwards and decode the best sequence
    match_sequence = []
    k = password.length - 1
    while k >= 0
      match = backpointers[k]
      if match
        match_sequence.push match
        k = match.i - 1
      else
        k -= 1
    match_sequence.reverse()
    #match_sequence就是不重叠的匹配列表，但是中间可能有没有覆盖的字符串

    # fill in the blanks between pattern matches with bruteforce "matches"
    # that way the match sequence fully covers the password:
    # match1.j == match2.i - 1 for every adjacent match1, match2.
    make_bruteforce_match = (i, j) =>
      pattern: 'bruteforce'
      i: i
      j: j
      token: password[i..j]
      entropy: @lg Math.pow(bruteforce_cardinality, j - i + 1)
      cardinality: bruteforce_cardinality
    # 使用bruteforce计算其中没有覆盖到的字符的entropy
    k = 0
    match_sequence_copy = []
    for match in match_sequence
      [i, j] = [match.i, match.j]
      if i - k > 0
        match_sequence_copy.push make_bruteforce_match(k, i - 1)
      k = j + 1
      match_sequence_copy.push match
    if k < password.length
      match_sequence_copy.push make_bruteforce_match(k, password.length - 1)
    match_sequence = match_sequence_copy

    min_entropy = up_to_k[password.length - 1] or 0  # or 0 corner case is for an empty password ''
    crack_time = @entropy_to_crack_time min_entropy

    # final result object
    password: password
    entropy: @round_to_x_digits(min_entropy, 3)
    match_sequence: match_sequence
    crack_time: @round_to_x_digits(crack_time, 3)
    crack_time_display: @display_time crack_time
    score: @crack_time_to_score crack_time

```

下面再看一下不同的匹配计算entropy的方式：

```coffee
  calc_entropy: (match) ->
    return match.entropy if match.entropy? # a match's entropy doesn't change. cache it.
    entropy_functions =
      dictionary: @dictionary_entropy
      spatial:    @spatial_entropy
      repeat:     @repeat_entropy
      sequence:   @sequence_entropy
      regex:      @regex_entropy
      date:       @date_entropy
    match.entropy = entropy_functions[match.pattern].call this, match
    
  dictionary_entropy: (match) ->
    ## 依赖于字典中字段的排位
    match.base_entropy = @lg match.rank # keep these as properties for display purposes
    match.uppercase_entropy = @extra_uppercase_entropy match
    match.reversed_entropy = match.reversed and 1 or 0
    match.l33t_entropy = @extra_l33t_entropy(match)
    match.base_entropy + match.uppercase_entropy + match.l33t_entropy + match.reversed_entropy

  START_UPPER: /^[A-Z][^A-Z]+$/
  END_UPPER: /^[^A-Z]+[A-Z]$/
  ALL_UPPER: /^[^a-z]+$/
  ALL_LOWER: /^[^A-Z]+$/

  extra_uppercase_entropy: (match) ->
    word = match.token
    return 0 if word.match @ALL_LOWER
    # a capitalized word is the most common capitalization scheme,
    # so it only doubles the search space (uncapitalized + capitalized): 1 extra bit of entropy.
    # allcaps and end-capitalized are common enough too, underestimate as 1 extra bit to be safe.
    for regex in [@START_UPPER, @END_UPPER, @ALL_UPPER]
      return 1 if word.match regex
    # otherwise calculate the number of ways to capitalize U+L uppercase+lowercase letters
    # with U uppercase letters or less. or, if there's more uppercase than lower (for eg. PASSwORD),
    # the number of ways to lowercase U+L letters with L lowercase letters or less.
    U = (chr for chr in word.split('') when chr.match /[A-Z]/).length
    L = (chr for chr in word.split('') when chr.match /[a-z]/).length
    possibilities = 0
    possibilities += @nCk(U + L, i) for i in [1..Math.min(U, L)]
    @lg possibilities

  extra_l33t_entropy: (match) ->
    return 0 if not match.l33t
    extra_entropy = 0
    for subbed, unsubbed of match.sub
      # lower-case match.token before calculating: capitalization shouldn't affect l33t calc.
      chrs = match.token.toLowerCase().split('')
      S = (chr for chr in chrs when chr == subbed).length   # num of subbed chars
      U = (chr for chr in chrs when chr == unsubbed).length # num of unsubbed chars
      if S == 0 or U == 0
        # for this sub, password is either fully subbed (444) or fully unsubbed (aaa)
        # treat that as doubling the space (attacker needs to try fully subbed chars in addition to
        # unsubbed.)
        extra_entropy += 1
      else
        # this case is similar to capitalization:
        # with aa44a, U = 3, S = 2, attacker needs to try unsubbed + one sub + two subs
        p = Math.min(U, S)
        possibilities = 0
        possibilities += @nCk(U + S, i) for i in [1..p]
        extra_entropy += @lg possibilities
    extra_entropy
```

其他匹配的计算方法就不罗列了，根据entropy计算score仅仅是数值上的变换，所以这里也略去了。

##4

我们再看一下计算字典匹配entropy的函数，其中计算base_entroy的公式可以再讨论一下。这种密码的列表很容易从Mark Burnett的网站中找到，甚至也可以自己统计现有泄露的密码库，但是转成这种列表之后实际上丢失了概率信息，仅仅保留了一个排名信息。

如果攻击者有几个完整的泄露密码库，他们还能从其中提取哪些有用的信息呢？我们如何利用这些信息评估未知密码的强度呢？

下一篇再谈。
