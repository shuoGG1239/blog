---
title: 做个纸片人live2d Viewer
date: 2020/01/31
categories: 
- 前端
tags:
- live2d
- electron
---

#### 展示下效果

* 双生最近的圣诞苏小真Live2d做的是真的好, 不得不说西山居的Live2d技术用得真溜!

![suxiaozhen_20191225_1.gif](https://s2.loli.net/2025/02/19/KNphZnWcXUl1SyT.gif)



#### 关于这个live2d Viewer

* 之前逛了一圈看很少有对live2d-v3做支持的, 大部分还是停留v2. 而最近双生这些高质量的live2d模型全是v3, 于是想自己做一个live2dViewer, 好把新老婆放桌面欣赏



* 技术选型上直接用electron了 (再见了老朋友Qt) , 因为未来打算在网站也复用一下, 所以选择electron无需犹豫



* 如何使用? 可以直接拉源码用electron跑, 后面我会打包执行文件到release直接用也行



#### 源码仓库

* 源码仓库: [https://github.com/shuoGG1239/live2d-Waifu](https://github.com/shuoGG1239/live2d-Waifu)

* 模型仓库: [https://github.com/shuoGG1239/Live2d-model](https://github.com/shuoGG1239/Live2d-model)



#### live2d 对接过程中的坑
* 一些游戏厂的模型, 并不按live2d cubisum的标准live2dParamId来制作, 例如双生的苏小真-圣诞Ver (id对比如下源码); 
我没搞懂为啥不按官方标准来定义, 这样只会徒增模型师和程序员的工作量
    - 如何获取模型的ParamId列表? 可以在framework的加载好模型后, 用idManager遍历和打印params

```typescript
// live2d官方默认的parameterid
/**
 * @brief パラメータIDのデフォルト値を保持する定数<br>
 *         デフォルト値の仕様は以下のマニュアルに基づく<br>
 *         https://docs.live2d.com/cubism-editor-manual/standard-parametor-list/
 */
export namespace Live2DCubismFramework
{
    // パーツID
    export const HitAreaPrefix: string = "HitArea";
    export const HitAreaHead: string = "Head";
    export const HitAreaBody: string = "Body";
    export const PartsIdCore: string = "Parts01Core";
    export const PartsArmPrefix: string = "Parts01Arm_";
    export const PartsArmLPrefix: string = "Parts01ArmL_";
    export const PartsArmRPrefix: string = "Parts01ArmR_";

    // パラメータID
    export const ParamAngleX: string = "ParamAngleX";
    export const ParamAngleY: string = "ParamAngleY";
    export const ParamAngleZ: string = "ParamAngleZ";
    export const ParamEyeLOpen: string = "ParamEyeLOpen";
    export const ParamEyeLSmile: string = "ParamEyeLSmile";
    export const ParamEyeROpen: string = "ParamEyeROpen";
    export const ParamEyeRSmile: string = "ParamEyeRSmile";
    export const ParamEyeBallX: string = "ParamEyeBallX";
    export const ParamEyeBallY: string = "ParamEyeBallY";
    export const ParamEyeBallForm: string = "ParamEyeBallForm";
    export const ParamBrowLY: string = "ParamBrowLY";
    export const ParamBrowRY: string = "ParamBrowRY";
    export const ParamBrowLX: string = "ParamBrowLX";
    export const ParamBrowRX: string = "ParamBrowRX";
    export const ParamBrowLAngle: string = "ParamBrowLAngle";
    export const ParamBrowRAngle: string = "ParamBrowRAngle";
    export const ParamBrowLForm: string = "ParamBrowLForm";
    export const ParamBrowRForm: string = "ParamBrowRForm";
    export const ParamMouthForm: string = "ParamMouthForm";
    export const ParamMouthOpenY: string = "ParamMouthOpenY";
    export const ParamCheek: string = "ParamCheek";
    export const ParamBodyAngleX: string = "ParamBodyAngleX";
    export const ParamBodyAngleY: string = "ParamBodyAngleY";
    export const ParamBodyAngleZ: string = "ParamBodyAngleZ";
    export const ParamBreath: string = "ParamBreath";
    export const ParamArmLA: string = "ParamArmLA";
    export const ParamArmRA: string = "ParamArmRA";
    export const ParamArmLB: string = "ParamArmLB";
    export const ParamArmRB: string = "ParamArmRB";
    export const ParamHandL: string = "ParamHandL";
    export const ParamHandR: string = "ParamHandR";
    export const ParamHairFront: string = "ParamHairFront";
    export const ParamHairSide: string = "ParamHairSide";
    export const ParamHairBack: string = "ParamHairBack";
    export const ParamHairFluffy: string = "ParamHairFluffy";
    export const ParamShoulderY: string = "ParamShoulderY";
    export const ParamBustX: string = "ParamBustX";
    export const ParamBustY: string = "ParamBustY";
    export const ParamBaseX: string = "ParamBaseX";
    export const ParamBaseY: string = "ParamBaseY";
    export const ParamNONE: string = "NONE:";
}


// 双生视界的parameterid (这游戏比较奇葩parameterid不按规矩来)
export namespace CafeGunGirlParam
{
    export const PartsArmPrefix: string = "PARTS_01_ARM_";
    export const PartsArmLPrefix: string = "PARTS_01_ARM_L_";
    export const PartsArmRPrefix: string = "PARTS_01_ARM_R_";

    export const ParamAngleX: string = "PARAM_ANGLE_X";
    export const ParamAngleY: string = "PARAM_ANGLE_Y";
    export const ParamAngleZ: string = "PARAM_ANGLE_Z";
    export const ParamEyeLOpen: string = "PARAM_EYE_L_OPEN";
    export const ParamEyeLSmile: string = "PARAM_EYE_L_SMILE";
    export const ParamEyeROpen: string = "PARAM_EYE_R_OPEN";
    export const ParamEyeRSmile: string = "PARAM_EYE_R_SMILE";
    export const ParamEyeBallX: string = "PARAM_EYE_BALL_X";
    export const ParamEyeBallY: string = "PARAM_EYE_BALL_Y";
    export const ParamBrowLY: string = "PARAM_BROW_L_Y";
    export const ParamBrowRY: string = "PARAM_BROW_R_Y";
    export const ParamBrowLX: string = "PARAM_BROW_L_X";
    export const ParamBrowRX: string = "PARAM_BROW_R_X";
    export const ParamBrowLAngle: string = "PARAM_BROW_L_ANGLE";
    export const ParamBrowRAngle: string = "PARAM_BROW_R_ANGLE";
    export const ParamBrowLForm: string = "PARAM_BROW_L_FORM";
    export const ParamBrowRForm: string = "PARAM_BROW_R_FORM";
    export const ParamMouthForm: string = "PARAM_MOUTH_FORM";
    export const ParamMouthOpenY: string = "PARAM_MOUTH_OPEN_Y";
    export const ParamCheek: string = "PARAM_CHEEK";
    export const ParamBodyAngleX: string = "PARAM_BODY_ANGLE_X";
    export const ParamBodyAngleY: string = "PARAM_BODY_ANGLE_Y";
    export const ParamBodyAngleZ: string = "PARAM_BODY_ANGLE_Z";
    export const ParamBreath: string = "PARAM_BREATH";
    export const ParamArmLA: string = "PARAM_ARM_L_A";
    export const ParamArmRA: string = "PARAM_ARM_R_A";
    export const ParamArmLB: string = "PARAM_ARM_L_B";
    export const ParamArmRB: string = "PARAM_ARM_R_B";
    export const ParamHandL: string = "PARAM_HAND_L";
    export const ParamHandR: string = "PARAM_HAND_R";
    export const ParamHairFront: string = "PARAM_HAIR_FRONT";
    export const ParamHairSide: string = "PARAM_HAIR_SIDE";
    export const ParamHairBack: string = "PARAM_HAIR_BACK";
    export const ParamHairFluffy: string = "PARAM_HAIR_FLUFFY";
    export const ParamShoulderY: string = "PARAM_SHOULDER_Y";
    export const ParamBustX: string = "PARAM_BUST_X";
    export const ParamBustY: string = "PARAM_BUST_Y";
    export const ParamBaseX: string = "PARAM_BASE_X";
    export const ParamBaseY: string = "PARAM_BASE_Y";
    export const ParamNONE: string = "NONE:";
}
```

* 由于部分游戏厂的模型不按标准来定义id, 就意味着想做通用live2dViewer就需要各种适配, 就挺坑的


* 还有live2d cubisum的sdk和framework, 文档和源码注释全是日文, 难道就不重视下岛外市场吗?
