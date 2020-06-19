---
title: Py小玩具-简单好用的OCR
date: 2018/07/21
---
#### 效果如下
* 截图识别
![img](https://i.loli.net/2018/07/21/5b528fab7fcbb.gif)
* 图片识别
![img](https://i.loli.net/2018/07/21/5b529366aa7c0.gif)

#### 代码
[https://github.com/shuoGG1239/Image2Text](https://github.com/shuoGG1239/Image2Text)

#### 介绍
* 本来一开始是用谷歌的tesseract, 也搞来了据说比较靠谱的trainedData, 但实际识别准确率实在不行, 于是放弃了, 不过这种好处就是可以离线识图, 只要能搞到靠谱好用的trainedData肯定是比在线识图要好啦
* 这个小工具最终是用了百度AI的OCR接口, 识别率不错, 特别是中文
* 百度AI的普通文字识别的接口一天最多只能免费调用500次/账号, 也不保证并发量, 想变强就充钱吧
* 识图的核心代码在上面代码仓库的[ocr_util.py](https://github.com/shuoGG1239/Image2Text/blob/master/ocr_util.py), 要注意的是API_KEY和SECRET_KEY的这两个变量要填上自己的key, 这俩key获取直接得到百度AI去拿, 直接登陆控制台-->鼠标移到右上角头像上-->安全认证--> AccessKey, 当然也可以直接用我的Key, 我都直接丢代码里, 但尽量用自己的吧, 说不定某天我不小心把AccessKey删了呢 :P
* Gui用的是PyQt5, 只支持python3, 所以python2.7的同学可以无视Gui部分...