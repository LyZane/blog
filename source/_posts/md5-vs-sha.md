---
title: MD5 vs SHA
date: 2022-04-11 21:12:15
tags:
---

本文将搞清楚一个问题：
> MD5 与 SHA 两种算法分别合适在什么情况下使用？

<br>

## 结论

1. 如果是防篡改场景，建议使用 MD5，因为它计算快速，摘要长度短。
2. 如果是防偷窥场景，必须使用 SHA-256 或更大长度的摘要算法。
3. 为避免彩虹表攻击，必须加盐。


<br>

## 摘要长度的差异

摘要越长，意味着 hash 范围越大，产生碰撞的概率越低，但是更长的摘要通常需要更多的计算时间和储存成本。

各种算法产生的摘要示例：

- `MD5:` fcd34772920cf22d3e8b9a7466e33e4f
- `SHA-1:` 768725e2f1576b240f82cde819ef91545af6afd1
- `SHA-256:` ff5b185c2eb56587a40eb547945753ce24a165abbc6f7b2b9cc76a665b9dd984
- `SHA-384:` 7e5d3277d43033d9252c932931c83c02329babac271b1e7a14e7a0bfa80e32932b6551eec56727e107f143f2d3ee507f
- `SHA-512:` 54134f6390362cfd748cc28549e98aef0f3403502b0a11558dce9f80f966e862b8fc9177451baeafdaa62a847072350bb6218ef1e32fba3fa6e781d966a14ff3

| 算法    | bit | byte | char |
| ------- | --- | ---- | ---- |
| MD5     | 128 | 16   | 32   |
| SHA-1   | 160 | 20   | 40   |
| SHA-256 | 256 | 32   | 64   |
| SHA-384 | 384 | 48   | 96   |
| SHA-512 | 512 | 64   | 128  |

<br>

## 计算耗时的差异

我使用 MacBook Pro (13-inch, M1, 2020) 进行了测试，对一个长度为 36 的字符串进行摘要计算，记录各算法执行百万、千万、亿次的耗时。

并根据这些结果大概估算了毫秒内计算次数，这样可以比较直观地比对它们的性能。

| 算法    | 1_000_000 | 10_000_000 | 100_000_000 | 次/毫秒 |
| ------- | --------- | ---------- | ----------- | ------- |
| MD5     | 196ms     | 1637ms     | 10441ms     | 6929    |
| SHA-1   | 164ms     | 1404ms     | 11399ms     | 7330    |
| SHA-256 | 353ms     | 3380ms     | 34847ms     | 2886    |
| SHA-384 | 382ms     | 3849ms     | 35902ms     | 2666    |
| SHA-512 | 359ms     | 3596ms     | 35644ms     | 2790    |

    结论：
    1. MD5 与 SHA-1 的性能相当
    2. SHA-256、SHA-384、SHA-512 的性能相当

以下是执行性能测试的代码：
```java
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.UUID;

public class Md5VsSha {
    public static void main(String[] args) throws NoSuchAlgorithmException {
        byte[] strBytes = UUID.randomUUID().toString().getBytes(StandardCharsets.UTF_8);
        int iterations = 1_000_000;

        System.out.println("iterations: " + iterations);

        String[] algorithms = {"MD5", "SHA-1", "SHA-256", "SHA-384", "SHA-512"};
        for (String algorithm : algorithms) {
            long spend = run(strBytes, algorithm, iterations);
            System.out.println(algorithm + " spend:" + spend + "ms");
        }

    }

    private static long run(byte[] bytes, String algorithm, int iterations) throws NoSuchAlgorithmException {

        long start = System.currentTimeMillis();
        for (int i = 0; i < iterations; i++) {
            MessageDigest digest = MessageDigest.getInstance(algorithm);
            digest.update(bytes);
        }
        return System.currentTimeMillis() - start;
    }
}
```

<br>

## 安全性的差异

摘要算法通常在安全领域的两个场景中被使用：防篡改、防偷窥。

**防篡改举例：**
sender 在发布文件的同时发布文件的摘要，receiver 在收到文件后同样对文件进行摘要计算，并与 sender 发布的摘要进行比对，以此判断文件是否为完整的原始文件。


**防偷窥举例：**
系统中用户登录账号的密码，对其进行 hash 后储存到数据库，这样即便是数据库泄露，储存在数据库中的 hash 串也不会暴露用户的明文密码。

<br>

在搜索了关于 MD5 与 SHA 算法的攻击方法后，总结出一共有两种攻击方法。

参照：
* https://zh.wikipedia.org/wiki/MD5
* https://3.14.by/en/md5
* https://wikichi.icu/wiki/MD5

1. 使用 GPU 快速生成大量摘要进行碰撞，也就是暴力破解。
2. 通过数学层面的破解，使得在已知摘要结果的情况下，能够使用普通计算机设备在几秒内找到另一个具有相同摘要的明文。


<br>


### 攻防

    首先要说明的是，摘要算法是一种信息有损算法，不论原始信息长度是否大于摘要长度，都不可能根据摘要信息还原出明文。

构建一个有意义的明文且与修改前的摘要值相同，这几乎不可能，所以摘要算法在防篡改场景是安全的，攻防主要在防偷窥场景中展开。

接下来以登录场景为例，攻击者获取到了数据库中的用户名和密码摘要，尝试登录系统，我们推演一下如何进攻与防守。

这里的攻击是指碰撞攻击，即找到另一个拥有相同摘要的明文，这个新的明文由于与真实密码具有相同的摘要，导致攻击者登录成功。

**【攻·暴力破解】**
攻击者如果使用暴力破解，以摘要长度相对较短的 MD5 为例，其摘要的可能性有 340282366920938463463374607431768211454 种，按目前较为先进的摘要生成速度 3.5亿/秒 计算，最差情况需要 308293802023029亿年。

考虑到需要储存一些状态来保障不生成重复的摘要，即便是在运气极佳的情况下，也很难在有生之年完成暴力破解，暴力破解失败。


**【攻·彩虹表】**
由于大家通常使用较短的有意义字符串作为密码，所以攻击者把所有可能的常用字符串全部计算一遍摘要，然后使用在数据库盗取的摘要进行比对，这样可以快速找到原始明文。

这种攻击方式被称之为彩虹表攻击，有些提供 MD5 查询的网站目前储存的键值对已经达到90万亿条，覆盖了几乎全部的常用密码，攻击成功。

**【防·组合 hash】**
彩虹表攻击手段基于“常用”这个特点极大地缩小了暴力破解的范围，所以我们可以相应地采取一些措施让这些密码变得不常见。

* md5(sha1(password))
* md5(md5(salt) + md5(password))
* sha1(sha1(password))
* sha1(str_rot13(password + salt))
* md5(sha1(md5(md5(password) + sha1(password)) + md5(password)))

使用组合 hash，或者将这些 hash 方式迭代很多次，如果攻击者不知道系统使用的哪种哈希函数，那么也就很难预先为这种组合构造出彩虹表，于是破解起来会花费更多的时间。

但这就安全了吗？

根据“柯克霍夫原则”：
> 即使密码系统的任何细节已为人悉知，只要密匙未泄漏，它也应该是安全的。

有时候攻击者很容易就能拿到源码，例如开源软件，或者对程序进行逆向，甚至内部人员在了解算法的情况下实施攻击。

即便不了解源码，通过系统中取出的一些密码与哈希值对应关系，很容易反向推导出加密算法。

<br>

**【防·加盐】**
如果在明文中掺入随机字符串之后再进行摘要，由于其原始明文在掺杂之后足够随机，所以加盐可以抵御彩虹表攻击。

关于盐值，有三个方面需要注意：
1. 盐的长度不能太短，太短的盐复杂度不够，建议使用与摘要长度相同或更长的盐。
2. 盐值不能复用，尽量每次 hash 掺入不同的盐。
3. 盐值要足够随机，可以考虑每次都生成一个 UUID 作为盐。


**【攻·数学破解】**

攻防进展到现在，暴力破解路线已经完败，轮到数学破解路线登场。

MD5 和 SHA-1 最初由中国的王小云院士破解，后来中国科学院的谢涛和冯登国优化了 MD5 破解方案，攻击者在已知摘要的情况下，使用普通计算机在几秒内即可计算出另一个具有相同摘要的 MD5 明文。

这意味着之前使用的组合 hash 和加盐的防御手段通通失效，好在我们还有 SHA-256、SHA-384、SHA-512 可以使用。

**【总结】**
1. 如果是防篡改场景，建议使用 MD5，因为它计算快速，摘要长度短。
2. 如果是防偷窥场景，必须使用 SHA-256 或更大长度的摘要算法。
3. 为避免彩虹表攻击，必须加盐。