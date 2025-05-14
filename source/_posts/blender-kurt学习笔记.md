---
title: bender-kurt教程学习笔记
date: 2024/12/09
categories: 
- 美术
tags:
- blender
---

### 前言
* 本文是学习`Kurt-blender教程`时的学习笔记, 主要记了些重点, 方便未来翻看回顾
* blender版本为4.3




### 0. 资料

* [Blender官方手册](https://docs.blender.org/manual/zh-hans/latest/sculpt_paint/navigation.html)



### 1. 基础操作

* 左边的工具可以长按出现子工具选项

* **视角**:

  * 中键拖动: 自由视角转换
  * Alt+中键拖动: 正视角转换 (Num1-8平替)
  * shift+中键拖动: 平移视角转换
  * Alt+Z: 透视模式
  * /: 进入物体的放大视图
  * N: 右边的插件栏是否显示
  * F12: 渲染预览

  

* **游标**(3D游标):

  * Shift+右键: 设置游标到当前鼠标位置
  * Shift+S: 弹出游标轮盘
  * Shift+C: 游标恢复至世界中心



* **原点**:

  * 原点是物体的根基, 1个原点就是1个物体; 如物体模式下复制, 原点数会+1, 编辑模式下复制, 原点数不变
  * 编辑原点: ![blender1_1741498163972.png](https://s2.loli.net/2025/03/13/bEZVvpnsf7AtQGr.png)

  

* **轴心点**:

  * ![blender1_1741497624120.png](https://s2.loli.net/2025/03/13/sAtGdgIyxTe7q2o.png)

* **变换**:

  * G(grab): 进入平移状态, 按XYZ固定轴方向, 左键确定右键取消
  * R(rotate): 进入旋转状态, 按XYZ固定轴方向, 左键确定右键取消 (期间填角度可以指定角度)
  * S(scale): 进入缩放状态, 按XYZ固定轴方向, 左键确定右键取消
  * Alt+G/R/S: 重置G,R,S状态, 也就是右上角的变换归0或归1
  * Ctrl+A: 应用(缩放); 应用后坐标参数会矫正, 所以物体模式下一定要记得ctrl+A!
  * ![blender1_1741427107220.png](https://s2.loli.net/2025/03/13/ZHouUz5wYgBO4Gk.png)

  

* **物体操作**

  * X: 弹出删除菜单

  * Shift+A: 新增模型或其他东西

  * Shift+H: 将选中物体外的东西隐藏

  * Shift+D: 复制选中物体
    - Alt+D: 关联复制选中物体(改一个同步给全部)

  * Ctrl+J: 合并选中的多个物体

  * Ctrl+L: 选中2个以上时, 按下弹出关联菜单(如关联材质等)

  * Ctrl+C,Ctrl+V: 复制粘贴物

  * Ctrl按住: 临时吸附原点

  * Ctrl+2: 添加细分修改器

  * 父子:

    * 绑父级: 选子物件(可多个)-> shift选父物件 -> Ctrl+P -> 物体: 绑定父子
    * 解父级: Alt+P 或 右键菜单父级, 有时根据情况选`清空父级保持变换`, 否则会飘

    


* **导航栏**

  * M(选中多个物体后): 移动到某个集合内, 当物体项距离集合太远时可用
    * .(句号): 导航栏快速定位到选中物体 
    * 父子级的选择, 可以手动shift把父子一个个点上; 或者在父级右键点选择层级:
    * ![blender1_1741622509603.png](https://s2.loli.net/2025/03/13/TevjquWkKlArXGh.png)

  

* **摄像机**

  * 将当前视图设置给摄像机 ![blender1_1741878914685.png](https://s2.loli.net/2025/03/13/LYdAWqvHx1ymSJD.png)

  



#### 1.1 总结

* 课程1.2一图流

![blender1_1741401842511.png](https://s2.loli.net/2025/03/13/UP84NwXcq3IWhBQ.png)

* 课程1.3一图流

![blender1_1741426646095.png](https://s2.loli.net/2025/03/13/Wru6RY89MlfgQkz.png)



### 2. 编辑模式

* Tab: 切换编辑模式或物体模式

* **编辑基础**
  * 1,2,3: 点,线,面
  * A: 全选 (取消全选是Alt+A)
  * L: 全选连接在一起的线
  * J: 连接顶点 (选择两个点后按下, 会连成一条边, 在两点之间存在面时使用)
  * F: 连接顶点 (选择两个点后按下, 会连成一条边, 在两点之间不存在面时使用)
  * E: 挤出; 记住在按下E后, 想取消不能按Esc, 而是要Ctrl+Z, 因为按下E时新的点线面就已提交!
  * M: 合并所选(如合并2个点成1点)
    * 例: 选2个点-> M -> 到末选节点: 合并2点
  * P: 分离 
    * 例: 树桩从树皮shift+D然后按P分离出苔藓, 此时树桩和苔藓为2个独立物体
  * Y: 拆分(不产生新物体,相对P)
  * V: 断开(点或边)
  * Ctrl+B(选中线时按): 在线条出细分出倒角, 滚轮控制细分数
  * Ctrl+E: 桥接循环边
  * Ctrl+X: 融并 (相当于安全的X, 干掉点或面时不会把面也干掉了)
    * 例: 选边-> X -> 融并边: 去掉多余边 (Ctrl+X)
  * Alt+选边: 选中周围的循环边(如多边环)
  * U: uv菜单
  * GG: 使点贴着物体线位移
  * Shift+鼠标移动操作: 精准缓慢
  * Ctrl+鼠标移动与操作: 吸附网格 

  

* **实用操作流**

  * 旋转90°: R->S->90
  * 水平对齐选中点: S->Z->0
  * 快速应用所有修改器: 右键->  转换为 -> 网格 (物体模式下): 快速应用所有修改器

* 视图着色方式修改, 为了方便区分可以改为随机
  * ![blender1_1741417327765.png](https://s2.loli.net/2025/03/13/AYgJ67PGHdkyViE.png)

* 对于多个点重合一起的情况可以: 网格->清理->按间距合并
    * ![blender1_merge_dots.jpg](https://s2.loli.net/2025/03/20/9fpnT5AYJRtF8zN.jpg)



#### 2.1修改器

* 曲线修改器

  * 创建流程: 新建曲线(路径曲线)-> 编辑曲线 -> 物体添加曲线修改器 -> 吸曲线

  * Alt+S: 修改半径
  * 通过曲线制作尾巴: 新建路径曲线-> 集合数据 -> 倒角 -> 深度拉大 -> Alt+S调各点的半径
    * ![blender1_1741506168212.png](https://s2.loli.net/2025/03/13/atRSgcV2ZB8MnfP.png)

* 蒙皮修改器

  * 创建流程: 创建平面-> 编辑 -> 全选点 -> M合并点 -> 创蒙皮修改器 -> 编辑E挤出点 -> 选点Ctrl+A调整蒙皮宽度
  * Ctrl+A: 修改半径
  * ![blender1_1741454621407.png](https://s2.loli.net/2025/03/13/fgavBRecl6Poh8V.png)

* 置换修改器
  * 石头示例: 立方体-> 细分修改器 -> 置换修改器 -> 纹理, 沃罗诺伊图 -> 调整参数
  * ![blender1_1741490598257.png](https://s2.loli.net/2025/03/13/TJ4WnOk26uCVGoU.png)



#### 2.2 总结

* 课程2.1一图流

![blender1_1741401984856.png](https://s2.loli.net/2025/03/13/Oikf5bhEaNFqGd9.png)



* 课程2.2一图流

![blender1_1741402007415.png](https://s2.loli.net/2025/03/13/u3Oe4JnVxsUtybG.png)





### 3. 着色器

* 着色器基础操作
  * Ctrl+右键划动: 切断材质连线
  * Ctrl+J (框选节点后): 给选中节点们组合成一个组
* NodeWrangler插件
  * Ctrl+T: 补全前置节点
  * Ctrl+Shift+左键: 快速把一个节点连接到输出预览
  * Ctrl+Shift+T (选择BSDF节点后): 选择材质4样, 自动补齐置换 (记得改材质置换模式为: 置换与凹凸)
    * 4样: color, displacement, normal, roughness
  * Ctrl+Shift+右键拉到另外一个节点: 为该2个节点创建一个混合BSDF节点
  * Alt+S (选混合节点后): 切换混合节点上下顺序
* UV绘制:
  * 进入`纹理绘制`模式可以给UV上色
  * 智能切UV: 编辑模式下-> A -> U -> 展开 -> 智能UV投射
  * 手动切UV: 编辑模式下-> 线模式下选好切割线(如Alt+线)-> U -> 标记缝合边-> A-> U-> 展开
  * ![blender1_1741526637025.png](https://s2.loli.net/2025/03/13/dCDpNH2q9o7lcm3.png)
  * 编辑模式下选好要遮罩的面后进入纹理绘制: ![blender1_1741526974381.png](https://s2.loli.net/2025/03/13/JSXUqa5Z3Mhp7gd.png)



#### 3.1 总结

![blender1_1741508086171.png](https://s2.loli.net/2025/03/13/etG1ODdx74W6cyV.png)



### 4. 粒子

* 节点组: 当只希望物体的一部分受影响, 可以用节点组, 有些修改器可指定节点组

![blender1_1741537847472.png](https://s2.loli.net/2025/03/13/fuVLk8iGzx52dZQ.png)

* 
* 变换跟时间轴绑定: ![blender1_1741538451050.png](https://s2.loli.net/2025/03/13/poUewnmAf48kjFT.png)
* 先创建一个圆环, 编辑模式下, F填充成圆盘, 然后物体选中圆盘后创建粒子系统![blender1_1741540346233.png](https://s2.loli.net/2025/03/13/lBk7UdN5iDxRCp2.png)





### 5. 场景灯光

* 导入
  * 关联: 无法修改, 但省内存 (需要去源文件修改,再重新加载)
  * 追加: 可以修改, 但消耗资源

![blender1_1741622089111.png](https://s2.loli.net/2025/03/13/HdjkpryQxJ5sch6.png)

* 摄像机
  * ![blender1_1741626491640.png](https://s2.loli.net/2025/03/13/tQvqHFVAPUma3M6.png)
  * 景深和透视比较常用![blender1_1741626585399.png](https://s2.loli.net/2025/03/13/m3ZsnL1d5EA6fK4.png)



### 6.动画

* **关键帧**
  * 插入关键帧: 选择物体后按`i`, 或者右键菜单中`插入关键帧`; 各种参数的右键菜单都有`插入关键帧`, 包括修改器里面的参数啥的都行
    * 选择物体后按`k`会弹出关键帧菜单, 相当于3.6版本的`i`的功能
  * 例如在缩放右键插入关键帧就仅在缩放下插帧![blender1_1741707671722.png](https://s2.loli.net/2025/03/13/XRyTYQA5tLaIgjq.png)
* **形态键**
  * 用于记录物体的某几个状态的形变, 例如下图的蝴蝶翅膀上下扇动
  * ![blender1_1741709628251.png](https://s2.loli.net/2025/03/13/87onABgChENyjVm.png)
* 曲线动画
  * ![blender1_1741709964727.png](https://s2.loli.net/2025/03/13/OPQg4RrMmd3Z9xk.png)
  * 添加修改器: 常用的有内建函数, 噪波, 循环
  * ![blender1_1741713017379.png](https://s2.loli.net/2025/03/13/QN4MUEc2niXDk5P.png)



* **骨骼**
  * 创建骨架: Shift+A -> 骨架
  * 编辑骨架: Tab进入编辑模式, 编辑方式和以前那些物体编辑基本一致; 挤出子骨骼用E, 右键细分可以分成n条小骨骼
  * 绑定骨架: 物体模式 -> 选物体 -> 选骨骼 -> Ctrl+P -> 附带自动权重 (如下图)
    * 绑骨骼前, 记得将先将物体清除父级, 不然容易出奇怪的bug
  * ![blender1_1741711669979.png](https://s2.loli.net/2025/03/13/CkwhBPLpQRjqfK4.png)
  * 骨骼带动物体: 进入`姿态模式`, 此时试试移动骨骼(G), 物体也会跟着动, 所以打关键帧也在姿态模式下打
    * 姿态模式下只允许操作骨骼, 属于骨骼的独占专属模式
  * ![blender1_1741712528677.png](https://s2.loli.net/2025/03/13/Te9jyDG3tplH2zA.png)
* 若骨骼在物体内部被挡住无法操作, 则可以在姿态模式下, `视图显示` -> `在前面`
  * ![3423475f-98aa-4e78-ae39-38e71ee1defd.png](https://s2.loli.net/2025/05/11/YznEgSd48hL9csX.png)
* 标准追踪
  * 追踪轴用的是局部轴, 如图狐狸头跟随蝴蝶的移动而转动
  * ![blender1_1741874470830.png](https://s2.loli.net/2025/03/13/MIHd6VApfJiU7zg.png)



### 7. 合成与输出

* 合成器
  * 合成器用于后期处理美化之类的, 相当于把Ae的部分功能搬进来, 属于纯2D的处理
  * ![blender1_1741875071761.png](https://s2.loli.net/2025/03/13/DWjmJB5z3SeUEPN.png)
* 输出
  * 主要关注如图几个点, 分辨率, 输出文件夹, 保存格式等;  建议保存为图片序列, 主要可以防止渲染一半崩了
  * ![blender1_1741875345050.png](https://s2.loli.net/2025/03/13/z5FXVIO96GTokhv.png)
  * 进行渲染输出文件 ![blender1_1741875518576.png](https://s2.loli.net/2025/03/13/5yFctROuNQvsapB.png)