---
layout: post
title: "蕙泉斋"
description: ""
category: 
tags: []
---
{% include JB/setup %}

## React源码剖析——生命周期

#### 组件和有限状态机

组件，通过状态渲染界面。组件具有自己的生命周期，在其生命周期的特定阶段，组件状态会发生改变，组件方法会被执行。

有限状态机(FSM)，即组件。表示具有有限个状态，并能在这些状态之间转移的模型。 

#### React的生命周期

自定义组件的生命周期，有三个状态：MOUNTING、RECEIVE_PROPS、UNMOUNTING。

**下图为React生命周期的三个状态**

![React生命周期](http://o7bm68198.bkt.clouddn.com/life_circle.png)

#### React的生命周期方法

React的生命周期中的三个状态，分别对应三个方法：mountComponent、updateComponent、unmountComponent。

这三个方法分别提供了两种处理方法：will方法在进入状态前调用；did方法在进入状态后调用。

将react-lifecycle mixin添加到需要观察的组件中，可以在控制台中观察到不同生命周期状态下，生命周期方法的调用顺序。反复实验后发现，首次装载组件时(First Render)、卸载组件时(Unmont)、重新装载组件时(Second Render)、组件状态更新时(Props Change/State Change)，生命周期方法的调用顺序如下。

![生命周期方法的调用顺序](http://o7bm68198.bkt.clouddn.com/life_circle_method.png)