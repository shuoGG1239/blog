---
title: bender雕刻学习笔记
date: 2024/12/29
categories: 
- 美术
tags:
- blender
---

### 前言
* blender版本为4.3



### 常用操作一图流

* 4.3之前的笔刷

![blender_dk3min.jpg](https://s2.loli.net/2025/03/23/w5eiGBtJKAE2lLO.jpg)

* 4.3之后的笔刷

![blender2_1742730538639.png](https://s2.loli.net/2025/03/23/XVNvrxuso3hGmZg.png)



### 模型切割

* 快速布尔: 选被切物体 -> 选形状物体 -> `Ctrl+Shift+-`
* 插件Carver:
  * Ctrl+Shift+X: 唤起插件
    * 不选物体唤起是创建模式, 选物体唤起是切割模式
  * 空格: 切换工具或确定切割



### 雕刻基础

* **笔刷**

  * 弹出笔刷菜单: Shift+空格

  * 调整笔刷大小: F或者`[`, `]`

  * 调整笔刷强度: Shift+F  (默认强度0.5太低了建议直接改成1.0)

  * 按Ctrl绘制: 可实现反向操作, 凸变凹, 凹变凸

  * 按Shift绘制: 可实现平滑

  * 常用几个笔刷:

  * ![blender2_1742738518947.png](https://s2.loli.net/2025/03/23/GbYyaWPdXeLlD6S.png)

  * 遮罩笔刷(M):

    * Ctrl: 按住进行绘制去除遮罩
    * Ctrl+i: 遮罩反向
    * Alt+M: 取消全部遮罩

    

* **动态拓扑**
  * 自动细分, 雕到哪细分到哪
  * 缺点: 由于不受控制, 所以可能导致面数过多
  * ![blender2_1742735146709.png](https://s2.loli.net/2025/03/23/doe5sGj7hY6v2QV.png)

  

  
* **重构网格**
  * 可以理解为雕刻模式下的细分, 不同细分下的雕刻效果不一样, 所以经常按需调整
  * Ctrl+R: 重构网格
  * R: 快速调整网格细分, R调整完再按Ctrl+R提交进行重构生效
  * ![blender2_1742735172396.png](https://s2.loli.net/2025/03/23/tZOu6ry34BsNlfI.png)

  
* **Quad Remesher插件**
  * 一键拓扑: 对所有面智能四边化处理
  * 使用前须关掉动态拓扑
  * ![blender2_1742736908457.png](https://s2.loli.net/2025/03/23/hqZy2dDcbmJvV9x.png)