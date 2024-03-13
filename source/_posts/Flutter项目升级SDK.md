---
title: Flutter项目更新SDK
date: 2023-11-02 10:51
tags: Flutter
---

最近把项目的SDK更新了，记录一下。我使用的是fvm作为版本管理。

1. 首先要去官网查询最新的版本列表以及Flutter和Dart对应关系。
[点击查看Flutter SDK 版本列表](https://flutter.cn/docs/development/tools/sdk/releases)
2. 使用`fvm install (flutter版本号)`
3. 使用`fvm list`查看SDK列表
4. 使用Android Studio打开项目，使用`fvm use (flutter版本号)`
5. 打开`pubspec.yaml`，更新下面的代码

```dart
environment:
     sdk: ">=3.0.5 <4.0.0"
     //这里是Dart 的版本号
```

6. 在`Android Studio`，在`Preferences | Languages & Frameworks` - `Dart`里更换`Dart`的SDK地址
7. 升级三方库的版本