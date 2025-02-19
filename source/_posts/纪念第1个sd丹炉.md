---
title: 纪念第1个sd丹炉
date: 2023/07/19
categories: 
- ai
tags:
- sd
---

#### 模型
* 当然, 这仅仅是训练一个sd的微调模型

* 想训练个画风, 但素材不多, 可预见的效果不会好到哪去
![shuogg1_train_images.jpg](https://s2.loli.net/2025/02/20/nMqwmgeJIBi2dPV.jpg)

* 核心的训练参数如下:
```text
network_module: lora
optimizer: AdamW8bit
repeat: 8
epoch: 20
network_dim: 32
unet_lr: 0.0001
text_encoder_lr: 1e-05
```


#### 效果
* 毫无意外的过拟合了
![shuogg1_out1.jpg](https://s2.loli.net/2025/02/20/5lYmfOVBcQeRrjD.jpg)
![shuogg1_out3.jpg](https://s2.loli.net/2025/02/20/X3ZUQO54Bk179gm.jpg)
![shuogg1_out2.jpg](https://s2.loli.net/2025/02/20/GClAS5DQosqnKrW.jpg)

