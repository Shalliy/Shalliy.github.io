---
title: Flutter难点解析
date: 2023-11-02 10:44
tags: Flutter
---

## 1.跨页面更新数据

在使用`GetX`的时候，有时候需要跨页面修改数据，比如有ABC三个页面，在`C`页面的某个值更改值时，需要在`A`页面体现出来，这时候就需要用到`Get.find<SomeController>();`。

但是如果`SomeController`没有注册，也就是没有调用`Get.put()`方法，就会发生`Crash`。这时候，我们需要判断`SomeController`是否注册过：

```dart
bool isControllerRegistered = GetInstance().isRegistered<SomeController>();
if (isControllerRegistered) {
    SomeController controller = Get.find<SomeController>();
    ...
}
```

## 2.onInit不调用的问题

使用`GetX`的时候，有时候`controller`的`onInit`方法不调用，原因很可能是在`view`中没有调用`controller`的方法，`controller`没有加载的缘故

## 3.TabBar的问题

如果使用`PageView`配合`BottomNavigationBarItem`，在进入App的时候，所有Page都会加载，有些时候这样做会产生问题。(比如b需要a的一些数据，但是b已经加载完成了，a的数据还没请求成功)。
所有最好是使用下列示例代码（不使用Pageview）：

```dart
Scaffold(
    appBar: appbar,
    body: pages[currentIndex],
    bottomNavigationBar: BottomNavigationBar(
        items: [
            item1,
            item2,
            ...
        ]
    )
);
```

## 4.Text组件显示红色字体，带有黄色下划线

这是因为该`Widget`没有被`Scaffold`、`Dialog`等包裹，因为这些组件内置`DefaultTextStyle`，如果你不想使用上述组件，也可以手动使用`DefaultTextStyle`包裹，这样`Text`就显示正常了。
