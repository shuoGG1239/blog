---
title: OTP动态口令及底层实现
date: 2020/06/06
categories: 
- 后端
---
#### 背景
* 最近用到了OTP, 遂mark一下
* 我们常用的那种倒计时验证码就是TOTP, 既不是叫OTP也不是叫MFA, 经常听有人这么说所以提一嘴

#### OTP
* 动态口令验证可以看作是服务端和客户端之间通过约定相同的算法来实现验证功能, 也即你在客户端看到的动态口令是客户端通过算法生成的无需请求服务端获取

#### TOTP
* 平时用的google动态口令用的就是TOTP(`Time-based One-Time Password`), TOTP基于HOTP
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

#### HOTP
* TOTP本质就是一个套皮HOTP, 核心是HOTP
* 我们从TOTP的视角来理解HOTP做了什么, 见代码和注释:
```go
type HOTP struct {
    secret []byte // 密钥, 每个人都有属于自己的一串密钥
    digits int    // 验证码的位数
}

/*
  counter就是`当前时间戳/设定的倒计时长`, return的就是平时看到的那串数字(通常为6位);
  eg:
  假设设定的倒计时长为60:
  counter为(1600000000 / 60)       返回 519365
  counter为((1600000000 + 5) / 60) 返回 519365
  counter为((1600000000 + 60) / 60)返回 797425
*/
func (h HOTP) At(counter uint64) string {
    counterBytes := make([]byte, 8)
    binary.BigEndian.PutUint64(counterBytes, counter)
    hash := hmac.New(sha1.New, h.secret)
    hash.Write(counterBytes)
    hs := hash.Sum(nil)
    offset := hs[19] & 0x0f
    binCodeBytes := make([]byte, 4)
    binCodeBytes[0] = hs[offset] & 0x7f
    binCodeBytes[1] = hs[offset+1] & 0xff
    binCodeBytes[2] = hs[offset+2] & 0xff
    binCodeBytes[3] = hs[offset+3] & 0xff
    binCode := binary.BigEndian.Uint32(binCodeBytes)
    mod := uint32(1)
    for i := 0; i < h.digits; i++ {
        mod *= 10
    }
    code := binCode % mod
    codeString := strconv.FormatUint(uint64(code), 10)
    if len(codeString) < h.digits {
        paddingByteLength := h.digits - len(codeString)
        paddingBytes := make([]byte, paddingByteLength)
        for i := 0; i < paddingByteLength; i++ {
            paddingBytes[i] = '0'
        }
        codeString = string(paddingBytes) + codeString
    }
    return codeString
}

// 验证阶段就简单了, 就是拿用户的输入code和上面的At结果对比
func (h HOTP) Verify(code string, counter uint64) bool {
    return h.At(counter) == code
}
```

#### 库
* [golang-gootp](https://github.com/gitchs/gootp)
* [python-pyotp](https://github.com/pyotp/pyotp)

#### 参考
* [RFC 4226 (HOTP)](https://tools.ietf.org/html/rfc4226)
* [RFC 6238 (TOTP)](https://tools.ietf.org/html/rfc6238)
* [动态令牌-(OTP,HOTP,TOTP)-基本原理](https://www.cnblogs.com/voipman/p/6216328.html)