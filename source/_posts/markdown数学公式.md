---
title: markdown数学公式
date: 2022/11/03
categories: 
- ai
tags:
- markdown
---

#### 一. Markdown公式基础关键字
* `$` 单行公式 `$`
* `$$` 多行公式  `$$`
* `\\` 公式内换行符
* `\!` 跟随
* `\,` 小空格
* `\;` 中空格
* `\`  大空格(约1字符)
* `\quad` 巨大空格(约2字符)
* `\qquad` 超巨大空格(约4字符)
* `_` 下标 `^`上标


#### 二. 基础符号
* 预览
$$
f(x) \\
x \pm y \\
x \times y \\
x \div y \\
x \cdot y \\
x^y \\
x_i \\
x_{i+1} \\
x^i_j \\
\dfrac{x}{y} \\
\sqrt{x} \\
x \gt y \\
x \lt y \\
x \ge y \\
x \le y \\
\cdots \\
\infty \\
\sin \\
\cos \\
\tan \\
\cot \\
\vec{x} \\
\sum_{x}^{y} \\
\prod_{x}^{y} \\
\lim_{x} \\
$$
* 对应的markdown
```
$$
f(x) \\
x \pm y \\
x \times y \\
x \div y \\
x \cdot y \\
x^y \\
x_i \\
x_{i+1} \\
x^i_j \\
\dfrac{x}{y} \\
\sqrt{x} \\
x \gt y \\
x \lt y \\
x \ge y \\
x \le y \\
\cdots \\
\infty \\
\sin \\
\cos \\
\tan \\
\cot \\
\vec{x} \\
\sum_{x}^{y} \\
\prod_{x}^{y} \\
\lim_{x} \\
$$
```

#### 三. 希腊字符
* 预览
$$
\alpha \
\beta \
\gamma \
\delta \
\epsilon \
\varepsilon \
\eta \
\theta \
\kappa \
\iota \
\zeta \
\lambda \
\pi \
\rho \
\xi \
\nu \
\upsilon \
\varphi \
\chi \
\psi \
\omega \
\Omega \
\Gamma \
\Delta \
\mu \
\Phi \
\nabla \
$$
* 对应的markdown
```
$$
\alpha \
\beta \
\gamma \
\delta \
\epsilon \
\varepsilon \
\eta \
\theta \
\kappa \
\iota \
\zeta \
\lambda \
\pi \
\rho \
\xi \
\nu \
\upsilon \
\varphi \
\chi \
\psi \
\omega \
\Omega \
\Gamma \
\Delta \
\mu \
\Phi \
\nabla \
$$
```

#### 四. 矩阵
* 预览
$$
 \left[
 \begin{matrix}
   1 & 2 & 3 \\
   4 & 5 & 6 \\
   7 & 8 & 9 \\
  \end{matrix}
  \right]
  \tag{这是注释}
$$
* 对应的markdown
```
$$
 \left[
 \begin{matrix}
   1 & 2 & 3 \\
   4 & 5 & 6 \\
   7 & 8 & 9 \\
  \end{matrix}
  \right]
  \tag{这是注释}
$$
```


#### 五. 表达式
* 预览
$$
\begin{cases}
x + y +  z \\
2x - 2y + 4z \\
-3x + 4y +5z
\end{cases}
$$
* 对应的markdown
```
$$
\begin{cases}
x + y +  z \\
2x - 2y + 4z \\
-3x + 4y +5z
\end{cases}
$$
```

#### 经典示例
* 麦克斯威方程
$$
\begin{array}{lll}
\nabla\times E &=& -\;\frac{\partial{B}}{\partial{t}} \
\nabla\times H &=& \frac{\partial{D}}{\partial{t}}+J \
\nabla\cdot D &=& \rho \
\nabla\cdot B &=& 0 \
\end{array} \
$$

