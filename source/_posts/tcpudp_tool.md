---
title: Py小玩具-Tcp&Udp&串口调试工具
date: 2019/03/31
categories: 
- PC端
tags:
- python
---
#### 简介
* 以前捣鼓嵌入式的时候写的, 就是那种网上搜就一大把的UDP+TCP+串口调试工具, 为了方便定制些功能于是自己搞了一个, 平时自己用, 功能没啥问题

#### 效果如下
* 界面
![normal.png](https://i.loli.net/2019/03/31/5ca0cfb5db3e3.png)
* 十分眼熟的那四种模式...Udp+TcpClient+TcpServer+串口通信
![4modes.png](https://i.loli.net/2019/03/31/5ca0cfc79fea9.png)

#### 代码
[https://github.com/shuoGG1239/TcpUdpSerialPortTool](https://github.com/shuoGG1239/TcpUdpSerialPortTool)

#### 补充
* 纯PyQt开发
* 界面右下角的`AOP`功能目的主要用于拦截发送和接收的数据, 然后在脚本中做额外处理, 一般场景用不到可无视...