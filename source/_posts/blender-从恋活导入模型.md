---
title: blender-从恋活导入模型
date: 2025/03/08
categories: 
- 美术
tags:
- blender
---

### 1. 插件下载

* KKBP提供了恋活导出插件(KKBP_Exporter)和blender导入插件(KKBP_Importer)
* KKBP:
  * 仓库: https://github.com/FlailingFog/KK-Blender-Porter-Pack
  * release: https://github.com/FlailingFog/KK-Blender-Porter-Pack/releases
  * wiki: https://kkbpwiki.github.io/
* 选哪个版本可以参照下图: 本次教程使用blender4.3, 所以用了7.2.2版本的KKBP
  * 表格来自kkbp-FQ: https://flailingfog.github.io/faq.html

![blender5_1743350980831.png](https://s2.loli.net/2025/03/31/AmG4XslZNOigWkM.png)



### 2. 插件安装

* KKBP_Importer是安装给blender的, 就走正常的本地插件安装流程就行
* KKBP_Exporter是安装给恋活的, 将里面的`KKBP_Exporter.dll`文件放在恋活目录中的`BepInEx/plugins`中



### 3. 模型导出

* 进入角色创建界面, 你会发现顶部多出的ui就是KKBP插件
* 选好要导出的角色, 勾选`Export Variations`和`Export Single Outfit`, 点`Export Model for KKBP`
* 等待一段时间后, 会弹出导出的模型文件夹, 里面有个`model.pmx`待会用于导入到blender

![blender5_1743351740890.png](https://s2.loli.net/2025/03/31/xYHVtoK5za9G6rS.png)



### 4. 模型导入

* 打开blender的KKBP插件, 点`导入模型`, 选上一步导出模型文件夹里的`model.pmx`
* ![blender5_1743352023571.png](https://s2.loli.net/2025/03/31/atM3pdNzTLgifxQ.png)
* 导入模型需要花费数分钟, 最终效果如图
* ![blender5_1743352153428.png](https://s2.loli.net/2025/03/31/1dI7XA2j9f8uwG5.png)