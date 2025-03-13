---
title: godot学习笔记-01 FoxyVsSlime
date: 2025/02/22
categories: 
- 游戏开发
tags:
- godot
---

#### 简介

* 本篇是制作FoxyVsSlime这个demo的学习笔记

* FoxyVsSlime的godot工程: [https://github.com/shuoGG1239/godot_play/tree/main/foxy_vs_slime](https://github.com/shuoGG1239/godot_play/tree/main/foxy_vs_slime)



#### 基础界面及操作

* [官方Manual: 2D界面](https://docs.godotengine.org/zh-cn/4.3/tutorials/2d/introduction_to_2d.html#d-workspace)

* 场景与节点

  * 下图1和图2都是给`main节点`添加子节点

  ![img_1740316128911.png](https://s2.loli.net/2025/02/23/cryvfN5nMFBL6DP.png)

![img_1740315945789.png](https://s2.loli.net/2025/02/23/Y9QxdBcybkfNjG1.png)

* 2D场景的对象编辑
  * 如图, QWES: 选择,移动,旋转,缩放

![img_1740315669627.png](https://s2.loli.net/2025/02/23/I9vMBqaPWYZsGF4.png)

* 文件系统(资源管理)
  * 资源直接从外面拖进来就行

![img_1740314995880.png](https://s2.loli.net/2025/02/23/D63gBEwKQnJaWHk.png)



* 检查器
  * 下图是从检查器查看和编辑`main节点`的属性

![img_1740316553935.png](https://s2.loli.net/2025/02/23/UIBNA1Esh2yQ4vJ.png)

* 脚本
  * 脚本是绑在节点上的, 节点实例化后会执行脚本上的各种handler如`_ready()`,`_process(float)`等等
  * @export可以将变量暴露到节点的检查器进行编辑, 例如绑定其他场景类或节点实例或资源等等
  * 脚本可以直接使用被绑节点实例的field, 不需要加this或self之类的, 查看修改都行, 例如main节点是Node2D类型, 有`position`属性, 可以直接`position=xxx`直接修改main节点的位置
  * 脚本对应节点上的子节点可以直接`$节点id`获取其节点
  * ![img_1740317436955.png](https://s2.loli.net/2025/02/23/rhEVwsut1JOc4HU.png)
  * 看下另外一个节点enemy, 是Area2D节点所以有碰撞信号如`_area_entered`, 红是信号, 绿是信号接受函数, 类似Qt的信号与槽
  * ![img_1740320846610.png](https://s2.loli.net/2025/02/23/xvTu2sHfIOZc5FX.png)
* 调试, 按F5运行游戏调试
  * 运行期间可以在`场景/远程`时实查看运行时节点
  * ![img_1740322422800.png](https://s2.loli.net/2025/02/23/Qrqm3otP9cuMOIK.png)



#### 关于position

* `position`变量是相对父节的位置, 全局位置则是`global_position`
* 节点默认的初始`position`就是`(0,0)`, 你可以看下Tranform的position或者在onready打印下看看
* 例如`game`和`player`是父子关系, `player`一开始放置在`game`的`(111,222)`位置, 此时如果`player`的`position`从`(0,0)`变为`(100,300)`, 他则在`game`的位置就变成`(211,522)`了 



#### FoxyVsSlime工程用到的类

* [官方Class Manual: Node2d](https://docs.godotengine.org/zh-cn/4.3/classes/class_node2d.html)

* 主场景: `Node2D`

  * 相机: `Camera2D`
  * 文字容器: `CanvasLayer`
    * 文字: `Label`
  * 背景: `Node2D` (Node2D可以当组来使用)
    * 背景图: `Sprite2D`
    * 边界: `StaticBody2D`
      * 物理碰撞区: `CollisionShape2D`
  * 定时器: `Timer `
  * BGM: `AudioStreamPlayer`

  

* 玩家foxy: `CharacterBody2D` (相比Sprite2D,Node2D多了`velocity`系列属性, 方便控制移动)

  * foxy动画雪碧图: `AnimatedSprite2D`
  * foxy物理碰撞区:  `CollisionShape2D`
  * foxy声音: `AudioStreamPlayer`
  * 定时器: `Timer`

  

* 敌人slime: `Area2D` (能检测其他带物理碰撞区的节点的碰撞信号`area_entered`,`body_entered`)

  * slime动画雪碧图: `AnimatedSprite2D`
  * slime物理碰撞区:  `CollisionShape2D`
  * slime声音: `AudioStreamPlayer`

  

* 玩家输入

  * `Input`: 单例
    * velocity = Input.get_vector("left", "right", "up", "down")



#### 其他常用类或函数

* `PackedScene`: 可以认为是Scene类的容器, 用instantiate()实例化

* `get_tree()`: 返回包含该节点的 SceneTree, 是`Node`的方法
* `get_tree().current_scene`: 主场景节点(即root的子节点main)
* `await get_tree().create_timer(1).timeout`: 同步等待1s
* `queue_free`: 删除本节点, 是`Node`的方法