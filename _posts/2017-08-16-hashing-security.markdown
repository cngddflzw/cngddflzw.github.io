---
layout: post
title:  "如何正确的使用 Hash 加密"
date:   2017-08-16
categories: hash security
---

原文来源: [hashing-security](https://crackstation.net/hashing-security.htm)

> **重要提示**
>
> 永远不要尝试实现自己的加密 Hash 函数, 使用现成的成熟的库来进行加密操作

# 什么是密码 Hash

```
hash("hello") = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
hash("hbllo") = 58756879c05c68dfac9866712fad6a93f8146f337a69afe7dd238f3364946366
hash("waltz") = c0e81794384491161f1777c232bc6bd9ec38f616560b120fda8e90f383853542
```

密码 Hash 主要是指对存在数据库中的密码不直接明文存储, 而使用加密后的加密函数存储, 这样在数据库暴露以后, 攻击者也无法知道真实密码是什么. 通常, 使用密码 Hash 的步骤如下:

> 1. 用户创建账号
> 2. 对用户密码进行 hash 并存入数据库
> 3. 用户登录, 输入明文密码, 对明文明文密码进行 hash 并与数据库中的密文密码进行比对
> 4. 如果比对成功, 则用户登录成功
需要注意的是, 这里说的 hash 函数, 与数据结构中所说的 hash 函数并不一样, 数据结构中所说的 hash 函数主要用于 hash table, 他追求的目的是高效和速度, 而不是安全. 只有 加密 hash 函数 (cryptographic hash functions) 才能用来实现密码 hash, 例如 SHA256, SHA512, RipeMD, WHIRLPOOL 等.

# 破解 Hash 密文

## 字典攻击 (Dictionary Attacks)

> 1. 字典攻击
> 2. Trying apple : failed
> 3. Trying blueberry : failed
> 4. Trying justinbeiber : failed
> 5. ...
> 6. Trying letmein : failed
> 7. Trying s3cr3t : success!

最简单的方法是使用字典攻击, 字典通常是由一些常用与密码的单词, 短语, 字符串以及他们对应的 hash 值预先生成的一个映射列表. 使用字典和数据库中的 hash 值比对, 如果匹配, 说明找到了明文密码. 通常明文密码会用缩写 (leet speak) 替代, 让字典更高效, 例如使用 "h3110" 替代 "hello".
彩虹表 (Rainbow Table) 
彩虹表是时间换空间 (time-memory trade-off) 的一种字典, 他牺牲了查询速度, 但是让字典变得更小. 彩虹表的原理不是本文重点, 有兴趣可参考 Rainbow Table

## 暴力破解 (Brute Force Attacks)

> 1. 暴力破解
> 2. Trying aaaa : failed
> 3. Trying aaab : failed
> 4. Trying aaac : failed
> 5. ...
> 6. Trying acdb : failed
> 7. Trying acdc : success!

暴力破解是将所有可能出现的字母组合使用特定 hash 函数进行 hash, 然后跟数据库的密码 hash 值进行匹配. 这种方法需要大量的 hash 计算, 需要的时间很长, 但是无法防御.

## 彩虹表 (Rainbow Table)

彩虹表跟字典类似, 但他对字典进行了优化, 通过时间来换取字典的空间 [参考](https://www.zhihu.com/question/19790488)

# 加盐 (Salt)

```
hash("hello") = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
hash("hello" + "QxLUF1bgIAdeQX") = 9e209040c863f84a31e719795b2577523954739fe5ed3b58a75cff2127075ed1
hash("hello" + "bv5PehSMfV11Cd") = d1d3ec2e6f20fd420d50e2642992841d8338a314b8ea157c9e18477aaef226ab
hash("hello" + "YYLmfY6IehjZMQ") = a49670c3c18b9e079b9cfaf51634f563dc8ae3070db2c4a8544305df1b60f007
```

普通的字典攻击只对使用和字典的 hash 加密方式相同的密文密码有效. 例如, 如果你对一个密码 hash 两次再存入数据库, 则相同的明文密码, 在字典和数据库中拥有不同的 hash 值, 这个字典就变得无效了.
你可以在密码的前面, 或者后面, 拼上一段随机的字符串后再进行 hash, 这段随机字符串就称为盐. 当你检查用户输入密码是否正确时, 也需要将盐拼上再进行 hash 来比对.
由于攻击者无法预先知道盐值是什么, 所以他无法预先构建字典, 这就让他们的攻击变得很低效.

## 错误的使用盐的方式

对一个加盐/加密方式是否安全的评估, 都是建立在攻击者可以获得你的盐值, 程序代码 (加密函数) 等这种最坏情况下

### 复用盐

复用盐是一种常见的错误的使用盐的方式. 攻击者仅需要在根据你的盐重新建立新的字典, 即可使用字典攻击破解你的密码.

### 短盐

如果你的盐太短, 攻击者可以为所有可能出现的盐的值建立一个字典. 例如你的盐只有 3 个 ASCII 码, 那么最多只需要建立 95 * 95 * 95 = 857,375 个字典即可. 所以我们建议盐尽可能长, 例如你使用 SHA256 进行加密, 则盐也应该有 32 个随机字节的长度.

# 正确的 Hash 方式

## 使用正确的盐

如上所说, 正确的盐应该是

> 1. 每个用户的盐都不同
> 2. 使用足够长的盐

## 总是在 Server 端加密

如果你是一个 Web 程序, 记住一定要在 Server 端进行加密. 因为:

> 1. 如果你的加密逻辑完全在客户端, 那么攻击者在拖库后, 直接用密文密码就可以登录你的服务, 甚至不需要知道你的原始明文密码.
> 2. 客户端 hash 加密并不是保护密码的方式, 因为中间人攻击的存在, 他甚至可以直接篡改你的 JS, 让加密函数无法运行. (这种攻击的防御方式应该是使用 HTTPS, 而非 hash)
> 3. 有的用户浏览器可能并未开启 JS 支持.

## 使用慢 hash 函数 (Slow Hash Functions)

盐保证了不能使用预先建立的字典来进行破解, 但他并不能抵御暴力破解. 而我们需要做的就是增加暴力破解的难度, 甚至让暴力破解称为不可能.
为了达到这个目的, 可以使用一些慢 Hash 函数对密码进行加密, 这样在进行暴力破解的时候效率会变得非常低. 例如使用 PBKDF2 或 bcrypt, 这些算法可以接收一个安全因子 (Security Factor) 作为参数, 安全因子的大小决定了函数的执行速度, 如果你的函数执行速度足够慢, 则暴力破解的成本将大大提高, 甚至变为不可能.
但是由于使用了慢 Hash 函数, 使得拒绝服务攻击 (Denial of Service, DoS) 成为可能, 所以需要做好 DoS 防御, 例如用户注册和用户登录功能加上验证码.
如果你担心慢 hash 函数成为程序的性能瓶颈, 你也可以将慢 hash 函数的运行放在客户端来做, 例如使用 [Stanford JavaScript Crypto Library](https://crypto.stanford.edu/sjcl/). 当然, 你依然需要在服务端再次进行 hash.
