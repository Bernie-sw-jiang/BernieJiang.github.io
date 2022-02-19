---
layout: post
title:  "Android属性动画(二)-Choreographer"
date:   2022-02-12
toc:  true
tags: [Android动画]
---
不常接触但是`Android`绘制链路上非常重要的类`Choreographer`其内部实现原理。

## 介绍

`Choreographer`负责接收显示系统的时间脉冲(垂直同步VSync信号)，并在时间脉冲到来时对其进行调整并已恒定的频率通知外部执行动画(Animations)、输入(Input)和绘制(Draw)操作，以此来保证这些操作的同步。App通常使用动画或`View`的更高层接口来间接与`Choreographer`交互。这些接口包括但不限于:

1. `ValueAnimation.start`
2. `View.postOnAnimation` / `View.postOnAnimationDelayed`
3. `View.invalidate` / `View.postInvalidateOnAnimation`

`Choreographer`中文翻译过来为舞蹈指挥，可以理解为`Choreographer`优雅地指挥三个UI操作同步跳舞，并呈现给用户。

## 源码分析

















参考

1. [Choreographer 解析](https://juejin.cn/post/6844903886940012551)

注

1. 以上代码均基于API 30
