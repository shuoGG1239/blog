---
title: llOnebot实现QQ机器人
date: 2024/10/24
categories: 
- 杂杂
---

#### QQ机器人的近乎完美方案
* 之前捣鼓qqbot结果让我非常失望, 各种功能限制, onebot这玩意就靠谱多了
* onebot缺点仅仅是平台限定windows


#### 部署
1. 安装`QQNT`: [安装包地址](https://dldir1.qq.com/qqfile/qq/QQNT/dd395162/QQ9.9.16.29456_x64.exe)
    - 在安装目录下找到`resources\app\app_launcher`, 到`app_launcher`目录下创建文件`llob.js`, 然后写入一句`require(String.raw`./LiteLoaderQQNT`)`
    - 在安装目录下找到`resources\app\app_launcher\package.json`, 将json里面的`main`对应的值改为"./app_launcher/llob.js"

2. 下载`LiteLoaderQQNT.zip`: [下载地址](https://ghp.ci/https://github.com/LiteLoaderQQNT/LiteLoaderQQNT/releases/download/1.2.3/LiteLoaderQQNT.zip)
    - 将解压出来的整个文件夹`LiteLoaderQQNT`复制到第1步的`llob.js`相同目录下
    - 在`LiteLoaderQQNT`文件夹内创建一个`plugins`文件夹

3. 下载`LLOneBot.zip`: [下载地址](https://ghp.ci/https://github.com/LLOneBot/LLOneBot/releases/download/v4.4.1/LLOneBot.zip)

4. 下载`dbghelp_x64.dll`: [下载地址](https://ghp.ci/https://github.com/LiteLoaderQQNT/QQNTFileVerifyPatch/releases/download/DllHijack_1.1.2/dbghelp_x64.dll)
    - 下载完后重命名为`dbghelp.dll`然后放到和`QQ.exe`同目录(就是第1步安装的那个QQ)

5. 此时打开第1步装的QQ, 上号, 在设置里面能看到`LLOneBot`就说明部署ok了


#### 机器人发消息
* 上面部署完之后就可以测试下了, 发消息的测试脚本如下:

```python
import json
import requests

headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept': 'application/json, text/plain, */*',
    'Upgrade-Insecure-Requests': '1',
    'Content-type': 'application/json',
    'Sec-Ch-Ua': '"Not_A Brand";v="8", "Chromium";v="120", "Google Chrome";v="120"',
    'Sec-Ch-Ua-Mobile': '?0',
    'Sec-Ch-Ua-Platform': 'Windows',
}


class OneBotApi(object):
    def __init__(self):
        self.addr = 'http://127.0.0.1:3000'

    def send_text_to_group(self, group_id: int, text: str):
        req = {
            "group_id": group_id,
            "message_type": "private",
            "message": [
                {
                    "type": "text",
                    "data": {
                        "text": text
                    }
                }
            ]
        }
        payload = json.dumps(req)
        resp = requests.post('%s/send_group_msg' % self.addr, data=payload, headers=headers)
        print(resp.text)

cli = OneBotApi()
your_group_id = 1145144444 # 这里填测试用的QQ群号
cli.send_text_to_group(your_group_id, 'hello')
```

#### 其他功能
* 直接看LLOneBot的官方文档即可, 非常全: [LLOneBot](https://apifox.com/apidoc/shared-a7b7a53c-8b85-4885-8573-103cfffb75ac/api-226391767)



#### 参考资料
* [LLOnebot教程](https://forum.itzdrli.cc/d/7-koishijiao-cheng-ru-he-shi-yong-liteloader-onebot-dui-jie-onebotsatori-shi-yong-jiao-cheng-qq-9915/8)