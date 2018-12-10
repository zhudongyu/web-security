[返回目录](./README.md)

## web 安全之 密码安全

### 一、密码的作用

    比对用户注册时留下的密码和用户登录的密码是否一样，以此作为一个凭证来证明“你”是“你”！

### 二、密码的存储

    (1) 密码存在哪些泄露风险？

        1.数据可被偷盗
        2.服务器被入侵
        3.通讯过程被窃听
        4.内部人员泄露数据
        5.来自其他网站的密码(俗称撞库)

    (2) 如何对泄露密码进行防御？

        首先我们必须对存储的密码进行加密防御，具体要求如下：

            1.严禁密码的铭文存储 - 防止泄露
            2.单项变换(原文密码变换成密文，而拿到密文无法算出原文密码) - 防止泄露
            3.变化复杂度要求 - 防止猜解
            4.密码复杂度要求 - 防止猜解
            5.加盐(随机字符) - 防止猜解
        
        既然密码需要加密处理，而且要求是复杂的单向变换，那么就需要一种算法来解决。而我们常用的算法是“哈希算法”，也叫“信息摘要算法”。
        具体特征如下：

            1.明文与密文一一对应
            2.雪崩效应(原文就算有一个字符不一样，那么密文就会完全不一样)
            3.从密文无法反推出明文
            4.密文为固定的长度，拥有32个字符
            5.常见的哈希算法：MD5 sha1 sha256

        虽然哈希算法后的密文无法被反推，但是市面上却拥有一种表叫做彩虹表，用来对抗哈希算法。根据密文查询彩虹表可以查出原文密码。
        具体彩虹表是什么，自行查阅，不做介绍。

        那么如何对抗彩虹表呢？

            1.帮助用户加强复杂度 - 加盐
              md5(userId + Date + 原始密码  + salt)

            2.变换次数越多越安全，其加密成本几乎不变，只是在生成密码的时候速度稍稍慢一些而已，但是却可以让彩虹表失效，因为数量太
              大，无法建立通用性。并且让解密的成本增大 N 倍。
              md5(sha1(sha256(userId + Date + 原始密码  + salt)))


### 三、密码加固代码层

    (1) 密码传输的安全性

        密码在 HTTP 传输过程中也是可以被窃听的，所以我们不能够进行明文传输，那么有什么办法可以限制呢？
        1.https 
        2.频率限制
            其作用是防止密码被猜解，也就是说限制用户在一段时间内输入密码的次数
        3.前端密码加密
            这里我们需要探讨下前端加密的意义？
            假定前端对密码进行了加密处理，而传输层被窃听了密码，但是攻击者不知道明文密码是多少，也就是用户的明文密码不会泄露，
            但是明文密码不泄露不代表着攻击者不能登录，也就是说前端传什么给后端进行登录的，那么攻击者同样传什么。不能阻止攻击
            者的登录也不能阻止他对用户身份的盗取，它仅仅能阻止用户的明文密码不被拿到。

            用户的明文密码不被拿到，又有什么意义呢？当然还是有意义的，就前面说的密码泄露的时候有一个重要的渠道(撞库)，我网站
            的密码没有泄露但是其他网站的密码泄露了，那有可能导致我网站的用户遭受损失，这个地方如果我们对前端的密码传输之前进
            行加密，能够防止用户的原始密码被泄露，防止用户原始密码泄露，自然就不可能这个密码再去登录其他网站，也就是最多这个
            用户的密码被泄露。

            所以前端密码进行加密意义有限，并不能阻止用户的资料会盗取或者阻止用户进行登录，可根据项目的实际情况进行选择，如果做
            了 HTTPS 那么前端加密其实可不做，但是如果对安全极度重视的话还是有必要做一下加密的。

    (2) 密码加密代码

        这里运用的语言是 JavaScript ，但是语义上是一致的。而且是前后端都采用了加密。

        我们先看下前端的代码：
```js
    var SUGAR = "adas123!@#!@#"
    password = md5(username + SUGAR + password)
```

        后端的代码：
```js
    // password.js - 定义密码相关功能
    var password = {}
    var crypto = require('crypto')
    // 定义 md5 函数
    var md5 = function (str) {
        var md5Hash = crypto.createHash('md5')
        md5Hash.update(str)
        return md5.digest('hex')
    }
    // 获取用户输入的密码
    password.getPasswordFromTxt = function (username, password) {
        var SUGAR = "adas123!@#!@#"
        return md5(username + SUGAR + password)
    }
    // 获取盐
    password.getSalt = function () {
        return md5( Math.random() * 99999 + "" + new Date().getTime() )
    }
    // 存储到数据库的密码
    password.encryptPassword = function (salt, password) {
        return md5(salt + 'asda!@#!' + password)
    }
    module.exports = password

    /* 
        后端验证登录密码 
        user -> user 表存储的数据对象
        data -> 用户登录传来的数据对象
    */
    // 如果用户没有salt 对用户密码进行加 salt 升级
    if(!user.salt) {
        var salt = password.getSalt()
        // 保持和前端一样的加密逻辑
        var newPassword = password.getPasswordFromTxt(user.username, user.password)
        // 数据库存储前进行加密
        var encryptedPassword = password.encryptPassword(salt,newPassword)
        // 存储到数据库
        await query(`update user set password = '${newPassword}', salt = '${salt}' where id = ${user.id}`)
        // 
        user.salt = salt 
        user.password = encryptedPassword
    }
    var enctrptPasswordFromUserInput = password.enctrptPassword(user.salt,data.password)
    if( enctrptPasswordFromUserInput !== user.password ) {
        throw new Error('密码不正确')
    }

```




              




            




