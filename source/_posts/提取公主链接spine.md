---
title: 提取公主链接spine
date: 2024/11/15
categories: 
- 美术
tags:
- spine
---

#### 简介

* 公主链接的Q版小人很适合当Spine的学习项目, 本篇介绍如何提取并转换为spine工程



#### 提取资源

1. 下载资源: 到 https://redive.estertion.win/spine/ 将`.altas, .png, .skel`都下载了

![princess_1740412159296.png](https://s2.loli.net/2025/02/25/Z4ry1kXLPHKeFQT.png)



2. 转json工程文件 (因为spine只认json不认skel): 到 https://naganeko.pages.dev/chibi-gif 点 `Add skeleton`, 将上步的3个文件选上提交, 然后到最下面的`Export skeleton as json for`点`Spine3.8`, 导出json文件

![princess_1740412345419.png](https://s2.loli.net/2025/02/25/KVdWq6EFsRZHT5D.png)



#### 导入资源到Spine

1. 到spine菜单,点 `导入数据`, 选择上一步导出的json文件

  ![princess_1740412422233.png](https://s2.loli.net/2025/02/25/YfcF31VOLJuHSBb.png)

2. 此时图片缺失, 到spine菜单, 点 `纹理解包器`, 解开atlas, 导出图片后, 将图片文件夹重命名为`Texture2D` 

![princess_1740412687373.png](https://s2.loli.net/2025/02/25/VqodWYEQcp56lOn.png)

3. 点右边的`图片`, 再点右下打开文件夹, 选择上步的`Texture2D文件夹`, 此时工程已经导入完成

![princess_1740412983855.png](https://s2.loli.net/2025/02/25/lt5N67paCsxyHGT.png)

4. 进入动画模式, 然后点各个动画的小点看看效果吧!

![princess_1740413117598.png](https://s2.loli.net/2025/02/25/j7u1nZTiKQ5rcla.png)





#### 最终Spine工程的结构

![princess_1740413225920.png](https://s2.loli.net/2025/02/25/d8CYEciyerIX37R.png)