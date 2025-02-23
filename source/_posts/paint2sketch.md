---
title: Py小玩具-绘图转线稿
date: 2019/06/30
categories: 
- ai
tags:
- Qt
---
#### 背景
* 临摹大佬的作品的时候丰富的颜色会反而产生莫名的干扰, 所以希望能转成干净的线稿!
*  PS的滤镜效果不理想, 也找不到其他合适的工具, 故自己撸一个试试

#### 源码
* [https://github.com/shuoGG1239/Paint2Sketch](https://github.com/shuoGG1239/Paint2Sketch)

#### 版本一(分支:no-ml)
* 原理就是直接用`opencv`各种骚操作, 效果如下图
* 右下角几个滑条用于调参, 参数主要有canny卷积核和开闭操作的几个阈值参数
* 效果就是这样...参数不管怎么调都很~辣~鸡

![img](/images/paint2sketch_remu_arg.png)
![img](/images/paint2sketch_sakura_arg.png)


#### 版本二(分支:master)
* 版本一的效果太辣鸡, 没啥思路, 弃坑搁置了一年; 无意瞎逛逛到了一个`lllyasviel/sketchKeras`的项目, 看似效果不错?
* `lllyasviel/sketchKeras`是走了神经网络的路线, 开源了test_code和训练好的mod, 总之先抄过来试试
* 效果如下: 牛逼!机器学习逆天改命!

![img](/images/paint2sketch_new_remu.jpg)
![img](/images/paint2sketch_new_chino.jpg)


#### 使用
* 如果想用的话直接clone下来, 再把release的mod.h5下载完丢根目录, 跑main.py就行了