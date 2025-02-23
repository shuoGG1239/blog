---
title: Py小玩具-罗马转假名
date: 2019/06/09
categories: 
- 前端
tags:
- python
- Qt
---
#### 罗马音转日文假名
* 罗马音转假名(平假片假), 简单好用...
* 属于个人需求, 偶尔要敲一段假名, (日文输入法太笨重污染桌面干净的右下角) 算是用的比较频繁的一个自制玩具

#### 效果
![japinput_1](/images/japinput_1.gif)


#### 使用
* 无需联网
* 除了长音符用了`yy`, 其余输入规则和主流的日文输入法差不多
* 例子
```text
ka か
wa わ
i  い
lo ぉ (小お)
le ぇ (小え)
nn ん
yy ー
```

#### 实现
* 纯PyQt5开发

#### 代码
* [https://github.com/shuoGG1239/JapInput](https://github.com/shuoGG1239/JapInput)