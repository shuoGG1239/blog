---
title: bender-恋活改模笔记
date: 2025/03/07
categories: 
- 美术
tags:
- blender
---



### 1. zipmod解包

* zipmod文件改后缀解压, 找到里面的.unity3d文件

* 用`SB3Utility`来解开`.unity3d文件`
  * 例如下图, 将`hyacine_shoes.unity3d`拖到SB3Utility, 找到对应的MeshRenderer后Export即可
  * 导出后.unity3d文件同目录下会导出对应的`.fbx`文件
  * ![blender4_1743179238426.png](https://s2.loli.net/2025/03/29/6zVyQ784oclASUE.png)





### 2. 导入Blender改模

* 直接导入生成的`.fbx`文件
* 导入后一般太小了, 为了方便修改, 放大100倍 (s->100)
* 此时你可以随意修改模型了
* 修改模型后, 导出为`.fbx`, 导出时物体类型为`骨架和网络`, 缩放为0.01, 因为上一步放大了100倍
  * ![blender4_1743180188931.png](https://s2.loli.net/2025/03/29/f3m74GTkEtXj8Nv.png)





### 3. 重新打包zipmod

* 再次用`SB3Utility`打开最开始待修改的那个`.unity3d`文件
* 将修改好的fbx拖入左下窗 -> 将模型拖到MeshRenderer的上一级 -> Rest/Bind -> 关闭fbx编辑
  * ![blender4_1743182702110.png](https://s2.loli.net/2025/03/29/WUz46gSR91GjQnA.png)
* 最后Ctrl+s, 保存.unity3d文件
* 重新打包成zipmod, 打包时记得压包根目录必须是`abdata`和`manifest.xml`