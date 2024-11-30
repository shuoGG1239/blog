---
title: QQ机器人qqbot把玩
date: 2024/07/30
categories: 
- 工具
tags:
- python
---

#### 关于qqbot的官方sdk
* 都2024年7月了, 有些sdk的最新更新时间还停留在去年, 最新的也就`python`版本, 这里我用的是`botpy`
* [https://github.com/tencent-connect/botpy](https://github.com/tencent-connect/botpy)


#### QQ群发图的问题
* `botpy`目前这个版本还没支持`QQ群`发base64图的功能, 这意味着必须先把图送到图床才行, 麻烦! 翻阅接口文档我发现其实接口上已经是支持了, 
但是所有语言的sdk没有一个支持就离谱

* 这时候需要魔改下`botpy`源码, 很简单, 找到`botpy/api.py`, 改下`post_group_file`这个方法, 加一个`file_data`的参数即可

```python
    async def post_group_file(
        self,
        group_openid: str,
        file_type: int,
        url: str,
        file_data: str,  # 新增该参数即可
        srv_send_msg: bool = False,
    ) -> message.Media:
        """
        上传/发送群聊图片

        Args:
          group_openid (str): 您要将消息发送到的群的 ID
          file_type (int): 媒体类型：1 图片png/jpg，2 视频mp4，3 语音silk，4 文件（暂不开放）
          url (str): 需要发送媒体资源的url
          srv_send_msg (bool): 设置 true 会直接发送消息到目标端，且会占用主动消息频次
        """
        payload = locals()
        payload.pop("self", None)
        route = Route("POST", "/v2/groups/{group_openid}/files", group_openid=group_openid)
        return await self._http.request(route, json=payload)
```

#### 吐槽下QQBot
* 咱开发这机器人无非就是要和QQ群里面的小伙伴们一起耍, 结果个人开发者开发的机器人只能在沙箱群里玩, 沙箱群最多就20人, 那还玩个蛋?
* 看官方文档和SDK的更新情况, 感觉这个项目大概是要凉了
* 希望有朝一日QQBot的产品能把它盘活了, 目前这玩意, 我是不想继续折腾了, 88


#### 最后贴个我的Demo供参考
```python
import botpy
from botpy import logging, Intents
from botpy.message import GroupMessage


class Bot(botpy.Client):
    img_gen_running = False
    app_id: int = 0
    app_secret: str = ''

    def __init__(self, intents: Intents):
        super().__init__(intents)
        self.app_id, self.app_secret = load_qqbot_cfg()
        tc_app_id, tc_app_secret = load_tecentcloud_cfg()
        tran = lang_translate.LanguageTranslater(tc_app_id, tc_app_secret) if tc_app_id != '' else None
        self.qq_msg_parser = qq_msg_parser.QQMsgParser(tran)
        self.ai_chat = ai_chat.AiChat(tc_app_id, tc_app_secret)

    async def on_group_at_message_create(self, message: GroupMessage):
        print(message.content)
        prompt_type, prompt = self.qq_msg_parser.msg_to_prompt(message.content)
        msg_req = 0
        if prompt_type == qq_msg_parser.PROMPT_TYPE_IMG:
            await self.resolve_img_gen(message, msg_req, prompt)
        else:
            # PROMPT_TYPE_TEXT case
            await self.resolve_chat(message, msg_req, prompt)

    async def resolve_img_gen(self, message: GroupMessage, msg_req: int, img_prompt: str):
        if self.img_gen_running:
            return self.group_reply_txt(message, msg_req, "aris手里的画没画完呐~等画完再发一遍给aris喵~")
        msg_req += 1
        await self.group_reply_txt(message, msg_req, "aris画得比较慢,要等半分钟喵~")
        msg_req += 1
        self.img_gen_running = True
        gen_code, file_or_msg = await img_gen.generate_img(img_prompt)
        self.img_gen_running = False
        _log.info(f'generate_img("{img_prompt}"): {gen_code} {file_or_msg}')
        if gen_code == 200:
            jpg_data = await img_gen.get_jpg_from_png(file_or_msg)
            jpg_data_b64 = base64.standard_b64encode(jpg_data).decode()
            await self.group_reply_image_data(message, msg_req, jpg_data_b64)
        else:
            await self.group_reply_txt(message, msg_req, "aris这张不想画了,换别的吧 T_T")

    async def resolve_chat(self, message: GroupMessage, msg_req: int, prompt: str):
        reply_txt = self.ai_chat.chat(prompt)
        await self.group_reply_txt(message, msg_req, reply_txt)

    async def group_reply_txt(self, message: GroupMessage, msg_req: int, txt: str):
        await self.api.post_group_message(
            group_openid=message.group_openid,
            msg_type=0,
            msg_id=message.id,
            msg_seq=msg_req,
            content=txt)

    async def group_reply_image_data(self, message: GroupMessage, msg_req: int, img_b64: str):
        upload_media = await self.api.post_group_file(
            group_openid=message.group_openid,
            file_type=1,
            url='',
            file_data=img_b64,
        )
        await self.api.post_group_message(
            group_openid=message.group_openid,
            msg_type=7,
            msg_id=message.id,
            msg_seq=msg_req,
            media=upload_media,
        )

    async def group_reply_image_url(self, message: GroupMessage, msg_req: int, file_url: str):
        upload_media = await self.api.post_group_file(
            group_openid=message.group_openid,
            file_type=1,
            url=file_url,
            file_data='',
        )
        await self.api.post_group_message(
            group_openid=message.group_openid,
            msg_type=7,
            msg_id=message.id,
            msg_seq=msg_req,
            media=upload_media,
        )
```
