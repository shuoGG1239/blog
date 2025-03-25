---
title: bender-MMD学习笔记
date: 2024/03/01
categories: 
- 美术
tags:
- blender
---

### MMD

* mmd主要定义了一套规范, 如骨骼命名规范, 有标准就可以使组件通用化, 如动作可以通用

* mmd文件
  * `.pmx`: mmd模型文件
  * `.vmd`: mmd动作(运动)文件
  * `.vpd`: mmd姿态文件



### MMD导入Blender

* 在MMD插件中: 模型导入-> 导入pmx文件-> 选择模型 -> 转换给Blender

![blender2_1742834248543.png](https://s2.loli.net/2025/03/26/hz3aSADMKlf1sVB.png)

* 若想单独编辑组件, 可按材质将各部件拆开, 如下图
  * 人物一般不建议做拆分, 会出现奇奇怪怪的问题

![blender2_1742834610107.png](https://s2.loli.net/2025/03/26/RaMeVzbEHvx3J8o.png)

* 模型描边: 点`边缘预览`可为模型描边
![blender2_1742835938862.png](https://s2.loli.net/2025/03/26/6dhmcRK13eH7Vga.png)



### 三渲二

1. 方式一: 直接用MiaoBox的三渲二插件

    * ![blender2_1742835239935.png](https://s2.loli.net/2025/03/26/WZE9RuD3JytQixc.png)
    * ![blender2_1742835466080.png](https://s2.loli.net/2025/03/26/juMbnp76C9YZNkX.png)
    * 效果如图: ![blender2_1742835541451.png](https://s2.loli.net/2025/03/26/J3BM2NFDQ5ZSwby.png)



### 动画

* 动作文件导入
  * vmd文件可以包含`表情动作`,`肢体动作`,`镜头动作`, 也可以分开包含
  * 假设vmd文件同时包含了以上3种动作:
    * 选中人物, 只能导入表情动作
    * 选中骨架, 只能导入肢体动作
    * 权限人物+骨架, 能同时导入表情+肢体动作
    * 选中摄像机, 只能导入镜头动作 (一般镜头动作是单独vmd文件)




* 动作导入示例
  * 全选模型 -> 运动/导入 -> 导入vmd文件, 边距改30(有多个文件就重复几次)-> 按空格看看动画效果
  * ![blender2_1742916179415.png](https://s2.loli.net/2025/03/26/1WVJgfpIHYLQidu.png)



* 导入动画后, 时间线不会显示关键帧的, 在选择骨骼后按i插入关键帧后就显示了

![blender2_1742837651081.png](https://s2.loli.net/2025/03/26/4UBzfVRGHmCWxkj.png)

* 摄像机导入
  * 选中摄像机再导入
  * ![blender2_1742920696893.png](https://s2.loli.net/2025/03/26/Qpl6BWhdRCHFVNZ.png)
  * 调整摄像机参数或手动K帧, 一般也是只调整这个摄像机
  * ![blender2_1742921097175.png](https://s2.loli.net/2025/03/26/pjSY8d79mgQT6HX.png)
* 表情动画
  * 表情动画一般是用形态键实现, 所以在形态键栏中调整或K帧
  * ![blender2_1742921691818.png](https://s2.loli.net/2025/03/26/4yCO9YQIpRwDqHW.png)



### 物理

* 物理烘培
  * 开启物理-> 更新世界 -> 烘焙 -> 等待物理烘培结束(图示例是烘到39帧)-> 空格看看动画效果
  * ![blender2_1742836752605.png](https://s2.loli.net/2025/03/26/keS3vjxUNCA9sgt.png)
* 刚体
  * 物理模拟的原理就是靠刚体模拟实现
  * 点击刚体图标显示刚体
  * 若要修改刚体: 删除烘焙 -> 关闭物理 -> 再去修改刚体
  * ![blender2_1742836997690.png](https://s2.loli.net/2025/03/26/oICvBaDT6bmz2h3.png)