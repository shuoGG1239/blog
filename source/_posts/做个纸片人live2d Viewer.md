---
title: 做个纸片人live2d Viewer
date: 2020/01/31
categories: 
- 前端
tags:
- live2d
- electron
---

#### 展示下效果

* 双生最近的圣诞苏小真Live2d做的是真的好, 不得不说西山居的Live2d技术用得真溜!

![suxiaozhen_20191225_1.gif](https://s2.loli.net/2025/02/19/KNphZnWcXUl1SyT.gif)



#### 关于这个live2d Viewer

* 之前逛了一圈看很少有对live2d-v3做支持的, 大部分还是停留v2. 而最近双生这些高质量的live2d模型全是v3, 于是想自己做一个live2dViewer, 好把新老婆放桌面欣赏



* 技术选型上直接用electron了 (再见了老朋友Qt) , 因为未来打算在网站也复用一下, 所以选择electron无需犹豫



* 如何使用? 可以直接拉源码用electron跑, 后面我会打包执行文件到release直接用也行



#### 源码仓库

* 源码仓库: [https://github.com/shuoGG1239/live2d-Waifu](https://github.com/shuoGG1239/live2d-Waifu)

* 模型仓库: [https://github.com/shuoGG1239/Live2d-model](https://github.com/shuoGG1239/Live2d-model)



#### live2d 对接过程中的坑
* live2d cubisum的sdk和framework, 文档和源码注释全是日文, 难道就不重视下岛外市场吗?



* 官方提供的live2d framework其实并不兼容双生的一些大制作模型, 例如苏小真-圣诞Ver, 需要修改很多地方, 比如大量的自定义Live2dParam, 还有各种事件, 虽然framework的代码量不多, 看完改改不难, 但是每个新模型都要适配一遍, 想做通用live2dViewer不得吐了!



* 目前live2d cubisum也没有立一些明文的规范, 特别是Live2dParam, 希望未来能做做标准化吧
