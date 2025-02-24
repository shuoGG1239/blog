---
title: spine学习笔记
date: 2024/11/13
categories: 
- 美术
tags:
- spine
---


#### 参考资料
* 官方教程简短可以刷一遍: [Spine官方Manual](https://zh.esotericsoftware.com/spine-skeletons)
* 企业级工程值得学习: [BlueArchive的Spine工程](https://github.com/respectZ/blue-archive-spine/tree/main/assets/spine)
* 本篇实例的原画文件: [https://www.live2d.com/zh-CHS/learn/sample/](https://www.live2d.com/zh-CHS/learn/sample/)



#### 新建工程

* 拆好件的psd, `菜单-->脚本-->PhotoshopToSpine-->OK`, 此时会输出1个带部件图的文件夹和1个json文件
* 打开spine, `主菜单-->导入数据-->选上一步的json文件`, 确定
* 此时spine工程已建好

![spine_1740328146481.png](https://s2.loli.net/2025/02/25/5XA6yCRZk8aQdul.png)



#### 基础操作

* 右键拖动: 移动视图, 相当于其他软件的按住空格
* 中键拖动: 框选
* 工具
  * 变换工具: `旋转, 移动, 缩放, 倾斜 (C, V, X, Z)`
  * 高级工具: `姿势, 权重, 创建 (B, G, N)`
  * 7个工具之间是互斥的, 同时只能使用1个工具

![spine_1740329193476.png](https://s2.loli.net/2025/02/25/Y8hEsMk9TGXbCN4.png)

* 锁
  * 当骨骼和部件绑一起时, 你只想调整骨骼的位置, 不想带动下面的部件一起变, 则可用`锁图片`实现, 如图
  * ![spine_1740331080569.png](https://s2.loli.net/2025/02/25/SlPhm6ug5otjxbk.png)



#### 骨骼

* 骨骼创建工具: 如下图
  * ![spine_1740328961670.png](https://s2.loli.net/2025/02/25/qD1A2RdxbpG438C.png)
* 骨骼同样可以`旋转, 移动, 缩放, 倾斜`, 它会带动骨骼下的子节点进行变换
  * ![spine_1740329583209.png](https://s2.loli.net/2025/02/25/EgPShMbw6vTd7cl.png)
* 快速创建骨骼
  * 在骨骼创建模式下, 点父骨骼(没得选就点root骨骼), 按住ctrl点待绑部件, 拖出新骨骼; 此时新骨骼的父子关系, 部件绑定关系, 还有命名都由系统自动安排好了 
  * ![spine_1740407644263.png](https://s2.loli.net/2025/02/25/u647ZBiHbQwSeAl.png)
* ik约束
  * 选2个骨骼, 创建ik约束
  * ![spine_1740408500663.png](https://s2.loli.net/2025/02/25/WNzceHkpaJoYwyI.png)
  * ![spine_1740408577437.png](https://s2.loli.net/2025/02/25/Ha8sdRQULj9AOfW.png)



#### 网格

* 点`编辑网格`, 若没有网格会自动创建; 然后点`生成`会自动布点, 但布点算法很垃圾!
* 网格的网点编辑就是`修改,创建,删除`来回切换使用 (难用得要死!)
  * ![spine_1740329856478.png](https://s2.loli.net/2025/02/25/BwrKSzHA4P2vCfJ.png)
* 选中网格, 点`绑定`, 再点左边的骨骼, 将`网格和骨骼做绑定`

  * ![spine_1740330138188.png](https://s2.loli.net/2025/02/25/ljSANR6YWXxuzZ3.png)
* 点高级工具的`权重`进入权重视图

  * ![spine_1740330187838.png](https://s2.loli.net/2025/02/25/oF6il7NCfH1BZdm.png)
* 随便新建个新骨骼`arm_r_bone2`, 还是选中上面刚做好的网格, 然后`绑定`这个新骨骼, 进入权重视图, 此时会有个权重笔刷, 改为`新增模式`, 刷下面几个点, 会发现变成紫色, 也就是说这几个点的控制权已转交给骨骼``arm_r_bone2``了
  * ![spine_1740330673184.png](https://s2.loli.net/2025/02/25/gbdMAJ5a9xr3mcC.png)



#### 动画

* 动画模式: 点击左上角切换进入
  * ![spine_1740331524833.png](https://s2.loli.net/2025/02/25/YtgIzB1iK34eLjn.png)
* K帧: 
  * 例如做一个手臂缩小的动画: `选中骨骼-->0帧处点缩放的钥匙--> 15帧处将缩放值调小(此时自动K帧)`
  * ![spine_1740331736403.png](https://s2.loli.net/2025/02/25/bdPJhTZVEUNHj67.png)