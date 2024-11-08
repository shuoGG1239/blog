---
title: 总结Pyinstaller的坑及终极解决方法
date: 2018/07/06
categories: 
- PC端
tags:
- python
---
#### 一. 首先要有个稳定环境
* 下面是博主经测试的觉得坑比较少的环境搭配
    1. Python3.4 + PyQt5.4 + Pyinstaller3.2.1
    2. Python3.5 + PyQt5.8 + Pyinstaller3.2.1

#### 二. Pyinstaller遇到坑没必要换打包工具
* 博主好几次用Pyinstaller遇到坑时都有考虑换工具如py2exe或cx-freeze之类的, 依旧无法解决 (最后还是用pyinstaller解决了)
* 所以没必要换其他工具, pyinstaller就够了 

#### 三. 坑1: 打包不了, 连exe都生成不出来

##### 解决方法
* 直接换Pyinstaller的版本, 即卸掉重装, 推荐用3.2.1

#### 四. 坑2: exe生成了, 但是跑不了
* 大多数情况都是被坑在这里

##### 解决方法
1. 遇到这种问题不管弹出什么样的错误提示, 在输出exe时参数加个'-d'即debug模式, 然后打开的时候能看到打印的错误信息了, 这招很好用
2. 留意一下程序依赖的一些资源文件, 检查下路径是否正确, 特别是程序里有相对路径的; 还有一些涉及到依赖系统默认资源的如默认字体啥的, 也得留意
3. 换下打包方式, 如onefile模式和onedir模式 (之前出现过onedir打包可以但onefile打包不行的情况)
4. 环境变量PATH中加上PyQt5的plugins的路径
5. 依旧不行则换个Pyinstaller的版本, 即卸掉重装, 推荐用3.2.1
6. 再不行则换操作系统试试, 有win10跑得了但到了win7就跑不了的情况 (弄个虚拟机测下找下问题在哪)


#### 五. 错误码集锦

##### main return -1
* 这种错误基本都是自己的问题, 只能在输出exe时参数加个'-d'即debug模式, 然后再查下打印的错误信息

##### Failed to execute script pyi_rth_pkgres
* 可以先换Pyinstaller的版本, 这个错误会消失, 但会弹出其他的错误信息, 然并卵
* 这种错误基本都是自己的问题, 只能在输出exe时参数加个'-d'即debug模式, 然后再查下打印的错误信息

##### Failed to execute script xxxx
* 这种错误基本都是自己的问题, 只能在输出exe时参数加个'-d'即debug模式, 然后再查下打印的错误信息

##### This application failed to start ... Qt platform plugin ...
* 这种错误先配下PyQt5的plugins的环境变量, 如博主的是C:\Python34\Lib\site-packages\PyQt5\plugins
* 不行再换Pyinstaller的版本 (貌似3.0.0这个版本有问题, 后来换3.2.1就没事了)

 