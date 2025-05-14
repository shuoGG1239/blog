---
title: astrbot源码
date: 2025/04/18
categories: 
- ai
tags:
- python
---

#### 前言
* 记录下, 防遗忘


#### 关于各阶段的handler
* 关键代码: star_handlers_registry.get_handlers_by_event_type

* 可以根据EventType找各个阶段的代码
```python
# astrbot/core/star/star_handler.py
class EventType(enum.Enum):
    """表示一个 AstrBot 内部事件的类型。如适配器消息事件、LLM 请求事件、发送消息前的事件等

    用于对 Handler 的职能分组。
    """

    OnAstrBotLoadedEvent = enum.auto()  # AstrBot 加载完成

    AdapterMessageEvent = enum.auto()  # 收到适配器发来的消息
    OnLLMRequestEvent = enum.auto()  # 收到 LLM 请求（可以是用户也可以是插件）
    OnLLMResponseEvent = enum.auto()  # LLM 响应后
    OnDecoratingResultEvent = enum.auto()  # 发送消息前
    OnCallingFuncToolEvent = enum.auto()  # 调用函数工具
    OnAfterMessageSentEvent = enum.auto()  # 发送消息后
```


#### llm_request的装饰器handlers
* astrbot/core/pipeline/process_stage/method/llm_request.py
```python
# 执行请求 LLM 前事件钩子。
# 装饰 system_prompt 等功能
handlers = star_handlers_registry.get_handlers_by_event_type(
    EventType.OnLLMRequestEvent
)
for handler in handlers:
    try:
        logger.debug(
            f"hook(on_llm_request) -> {star_map[handler.handler_module_path].name} - {handler.handler_name}"
        )
        await handler.handler(event, req) # 执行handler
```

* 完整调用链
    1. astrbot/core/pipeline/scheduler.py: await self._process_stages(event)
    2. astrbot/core/pipeline/scheduler.py: async for _ in coro
    3. astrbot/core/pipeline/process_stage/stage.py: async for _ in self.llm_request_sub_stage.process(event)
    4. astrbot/core/pipeline/process_stage/method/llm_request.py: for handler in handlers