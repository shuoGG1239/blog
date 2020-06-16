---
title: 简述OTP动态口令及其实现
date: 2020/06/06
---
#### 背景
* 最近用到了OTP, 遂mark一下

#### OTP
* 动态口令验证可以看作是服务端和客户端之间通过约定相同的算法来实现验证功能, 也即你在客户端看到的动态口令是客户端通过算法生成的无需请求服务端获取

#### TOTP
* 平时用的google动态口令用的就是TOTP(`Time-based One-Time Password`), TOTP基于HOTP, 所以弄懂TOTP即可
* 原理: 假设用的是30秒间隔的六位口令, 精简版伪代码:
```go
// secret为密码, timestamp为时间戳, 返回口令
GetOTPCode(secret, timestamp) {
    hs = hmac(secret, timestamp/30)
    // hsToInt是对hs这个[]byte进行各种&与偏移操作然后转为int
    intHs = hsToInt(hs) 
    code = intHs % 1000000
    return code
}
``` 

#### 库
* [golang-gootp](https://github.com/gitchs/gootp)
* [python-pyotp](https://github.com/pyotp/pyotp)

#### 参考
* [RFC 4226 (HOTP)](https://www.cnblogs.com/voipman/p/6216328.html)
* [RFC 6238 (TOTP)](https://tools.ietf.org/html/rfc6238)
* [动态令牌-(OTP,HOTP,TOTP)-基本原理](https://www.cnblogs.com/voipman/p/6216328.html)