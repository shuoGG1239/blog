---
title: Py小玩具-MarkDown图床助手
date: 2018/10/25
---
#### 简介
* 现在markdown越用越频繁了, md好用是好用, 但就是贴图片的时候有些麻烦: 要截图->上传图片->复制图片url, 于是做了个简单的工具: `截图->传图->生成图片url`三合一, 三...三分之一倍的快乐呀!


#### 效果
* 截图上传(也可以用快捷键`ctrl+shift+alt+F8`)

![img](https://i.loli.net/2018/10/25/5bd1b6bc0ce73.gif)

* 上传后自动将图片url复制到剪贴板, 直接粘贴即可

![img](https://i.loli.net/2018/10/25/5bd1b75390dec.gif)

* 也可以选择图片上传

![img](https://i.loli.net/2018/10/25/5bd1b786a5f92.gif)

* 也可以直接拖到框中上传

![img](https://i.loli.net/2018/10/25/5bd1b7c38eded.gif)



#### 相关地址
* exe下载: [https://github.com/shuoGG1239/PicPong/releases](https://github.com/shuoGG1239/PicPong/releases)
* 源码仓库: [https://github.com/shuoGG1239/PicPong](https://github.com/shuoGG1239/PicPong)



#### 技术相关
* 纯pyQt5开发
* 图床是用了`sm.ms`, 简单好用
* 后期也许会加一些其他的图床吧(也许...)