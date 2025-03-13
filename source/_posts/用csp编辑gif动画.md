---
title: 用csp编辑gif动画
date: 2025/03/01
categories: 
- 美术
---

#### 前言
* 以前编辑gif一直用`Ulead GIF Animator`, 说实话, 很不顺手, 想寻找新的工具
* 尝试用了csp的动画功能来做gif编辑, 发现意外的顺手!




#### 前置工作: gif转图片序列
* csp不接受gif的图片, 这点挺蠢的, 所以需要我们手动将gif转成图片序列
* 这里可以用脚本解决:
```python
from PIL import Image
import os
import sys

def gif_to_frames(gif_path, output_folder):
    gif = Image.open(gif_path)
    if not os.path.exists(output_folder):
        os.makedirs(output_folder)
    
    gif_name = os.path.splitext(os.path.basename(gif_path))[0]
    frame_number = 0
    try:
        while True:
            frame = gif.copy()
            frame.save(os.path.join(output_folder, f"{gif_name}_frame_{frame_number:04d}.png"), "PNG")
            frame_number += 1
            gif.seek(gif.tell() + 1)
    except EOFError:
        pass

gif_path = "your_gif_file_name.gif"
output_folder, _ = os.path.splitext(gif_path)
gif_to_frames(gif_path, output_folder)

```



#### gif导入到动画轨道

* 先创建一个`长宽大小和gif一致`且`边距上下左右均为0`的动画
* 打开上一步的动画序列图片的文件夹, 框选所需序列图片, 然后直接拖到csp的动画文件夹里, 如图:
  * ![csp2_1740815430566.png](https://s2.loli.net/2025/03/01/OnufgGcbmvReLaY.png)
  * 原先自带的`胶片`记得删掉, 如上图的文件夹最下面那个图层, 直接删掉就行
* 展开动画文件 -> 右键轨道标签 -> 批量指定胶片 - > 从现在的动画胶片名称进行指定 -> 确定
  * ![csp2_1740816023358.png](https://s2.loli.net/2025/03/01/DPNaAjr6cVFMTvX.png)
  * ![csp2_1740816136230.png](https://s2.loli.net/2025/03/01/DphKzLxeMIuJXP7.png)
* 此时已经导入完成, 可以试试播放下动画



#### 编辑动画

* 觉得图片太大了, 则可以缩小图片大小, `编辑`-> `图片大小`
* 觉得图片帧之间的间隔太小, 那只能手动一张一张往后面拖来实现
* 至于图片的修改, 直接点所要修改的图层(胶片)进行修改就行, 怎么涂都行



#### 导出动画

* 文件 -> 导出动画 -> Gif动画