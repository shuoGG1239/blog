---
title: markdown锚点跳转的坑
date: 2019/11/13
categories: 
- 工具
---
#### 背景
* 写markdown有这样的需求: 点击某个词跳转到markdown文章的某个位置(某个锚点), 但是写完发现有些点了跳不过去
* 原因就是跳转锚点的格式没写对, 格式见下面

#### 锚点title需要注意的格式
* 必须全小写
* 空格用'-'代替
* '_' '()'需要去掉

#### 错误例子
```
[点我跳转1](/shuogg/article.html#如何取一个好的ID)  
[点我跳转2](/shuogg/article.html#game system搭建)
[点我跳转3](/shuogg/article.html#game_system搭建)
```

#### 正确例子
```
[点我跳转1](/shuogg/article.html#如何取一个好的id)  
[点我跳转2](/shuogg/article.html#game-system搭建)
[点我跳转3](/shuogg/article.html#gamesystem搭建)
```