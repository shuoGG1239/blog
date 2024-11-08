---
title: Jetbrains Clion官方支持了Stm32的项目搭建, 说下感想
date: 2020/05/05
categories: 
- 工具
---
#### 背景
* 得知Clion 2019.1之后的版本官方直接支持Stm32项目的创建, 遂怀揣激动之心准备一试...
![img](/images/clionstm32.jpg) 

#### 吐槽
* 照着别人的教程, 一顿操作猛如虎, 一会捣鼓`OpenOCD`, 一会捣鼓`arm-none-eabi-gcc`... ...说实话, 过程挺麻烦的, 会遇到一些坑
* 手头上只有一块老stm32的核心板还有一个Jlink, 烧写调试也只能靠Jlink. 结果捣鼓了老半天, Jlink这块没办法打通, 即没办法用Jlink愉快地Debug, 遂放弃

#### 结论
* 现阶段还不完善, 该用keil的还是得用keil (当然也有可能只是我的搭建姿势有问题, 望指教)
* 期待未来某一天Clion能够拳打Keil脚踩IAR

#### 参考教程
* [Clion下开发STM32](https://www.jianshu.com/p/a3d529c208c9)
* [用clion自带的嵌入式开发功能和stm32cubeMX开发stm32!!!](https://blog.csdn.net/keysking/article/details/89511247)