---
title: View的Layout方法粗解
date: 2018-08-28 15:32:04
tags:
---

参考链接：https://www.jianshu.com/p/2ef48c2f0c97

    View的Layout方法主要有以下几种：
        layoutSubviews
        layoutIfNeeded
        setNeedsLayout
        setNeedsDisplay
        drawRect
        sizeThatFits
        sizeToFit
        
<!--More-->

### 我们先说结论：
* `layoutSubviews`方法不能手动调用，需要调用`[self setNeedsLayout]`来触发。
* `drawRect`也不能手动调用，需要调用`[self setNeedsDisplay]`来触发。
* 最后剩下`layoutIfNeeded`方法。这个方法在使用约束的时候调用，**可以立即更新效果(立即刷新)**。(例如：autoLayout 跟 frame 混用)`setNeedsLayout`不会立即刷新。
* `sizeThatFits:`会计算出最优的size，并返回给你，但是view本身并不会改变size。
* `sizeToFit`会计算出最优size，并且改变view自身的size

#### 1. `layoutSubviews`调用时机（系统调用）
* 直接调用`[self setNeedsLayout];` 。
* `initWithframe:`如果初始`frame`为`CGRectZero`则不调用，否则调用
* `addSubview`的时候。
* `view` 的 `size`发生改变的时候，**前提是`frame`的值设置前后发生变化**。
* 滑动`UIScrollView`的时候。
* 旋转`Screen`会触发父`View`上的`layoutSubviews`事件**(待验证，作者说验证过没有触发)**

#### 2. `setNeedsDisplay`调用时机
该方法在调用时，会自动调用`drawRect`方法。`drawRect`方法主要用来画图。
so：
当需要刷新布局时，用`setNeedsLayOut`方法；
当需要重新绘画时，调用`setNeedsDisplay`方法。


### 粗浅的总结：
总的来说：我们可以重写父类的`layoutSubviews` 和 `drawRect` 方法，但是我们不能手动调用它们，而是需要分别调用`setNeedsLayout` 和 `setNeedsDisplay`这两个方法来触发。


