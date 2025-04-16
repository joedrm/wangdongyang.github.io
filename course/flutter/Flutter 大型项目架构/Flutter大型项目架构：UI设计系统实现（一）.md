---
title: Flutter大型项目架构：UI设计系统实现（一）
date: 2024-04-21 17:07:56
tags:
---

![](https://s2.loli.net/2024/04/26/phrXKsvEDQwTHl1.png)

前面几篇讲了很多关于分层设计、状态管理和依赖管理，但是作为前端开发，设计资源怎么去管理、设计系统如何去实现其实在日常开发中接触是最多的，每个开发者或者项目都有一套自己的管理方式或实现方式，今天来分享一下我在大型项目中是如何做设计和实现资源管理的。

<!-- more -->

## `Flutter` 默认的设计系统

在 `Flutter`  写页面的时候通常会用到 `package:flutter/material.dart` 和 `package:flutter/cupertino.dart` ，主要是为了使用 `Flutter SDK` 提供的 `Material/Cupertino Design` 风格的UI组件和工具，这其中它的默认主题。虽然您可以自定义默认文本主题的标题样式，但被严格限制为 3 个级别：`Large`, `Medium`, `Small`， `Color` 的命名的变量个数也是有限制的。 

```Dart
import 'package:flutter/material.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter技术实践',
      theme: ThemeData(
        // 设置主色为蓝色
        primaryColor: Colors.blue,
        // 设置强调色为绿色
        accentColor: Colors.green,
        // 设置默认字体为Roboto
        fontFamily: 'Roboto',
        // 设置默认文本样式
        textTheme: TextTheme(
            // 设置标题文本样式
            displayMedium:
                TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
            // 设置正文文本样式
            bodyMedium: TextStyle(fontSize: 16),
        ),
        // 设置按钮的样式
        buttonTheme: ButtonThemeData(
          buttonColor: Colors.blue, // 按钮颜色
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(8), // 圆角半径
          ),
          textTheme: ButtonTextTheme.primary, // 按钮文本颜色
        ),
      ),
      home: HomePage(),
    );
  }
}
```
这些原生的设计资源看起来好像是够用的，但是当我们需要给 `ListView` 一个背景颜色时，没用合适字段去表示。而在大型的 `Flutter` 项目中这种情况加太多了，又该如何做呢？

## 定制自己的设计系统
设计系统是一套用于管理和共享设计资源的方法和工具集合。它包含了一系列的设计准则、组件、样式、布局规范等，将可重复使用的组件、样式指南（包括字体、颜色、尺寸等）和使用标准集合在一起集中去管理，旨在确保应用程序的视觉和交互一致性，并提高开发效率和设计协作能力。

对应到 `App` 中，将设计系统通常分为 3 类：
- 原子级别：如 `color`、`font`、`padding`、`radius` 等等，这是构成设计系统的基础。
- 分子级别：如 `button`、`checkboxes`、`radio boxes`、`dividers`、`input fields`，是一些最基本和常见的 `widget`。
- 细胞级别：如 `appbars`、`complex card`，甚至自定义 `widget`（如 `CustomPainter`），一些更复杂的 `widget`。

下面来实现一套自己的设计系统。

## `Theme Extension`
通常自定义一些常用的主题样式属性，会将其封装在 `ThemeExtension` 中，`ThemeExtension` 是对 `ThemeData` 类的扩展，用于简化主题样式的设置和访问，使用 `ThemeExtension` 轻松地定义和管理我们自己的主题样式，并在整个项目中访问和应用。

### `Color Theme`
我们先以 `Colors` 为例，创建一个名为 `AppColorsTheme` 的类，继承自 `ThemeExtension`，如下面代码:

```Dart
import 'package:flutter/material.dart';

class AppColorsTheme extends ThemeExtension<AppColorsTheme> {

  static const _colorB0B0B0 = Color(0xFFB0B0B0);
  static const _colorEFEFEF = Color(0xFFEFEFEF);
  static const _color333333 = Color(0xFF333333);

  static const _color6C6C6C = Color(0xFF6C6C6C);
  static const _color7D7D7D = Color(0xFF7d7d7d);
  static const _color676767 = Color(0xFF676767);
  
  // 页面中真正使用的颜色名称
  final Color backgroundDefault;
  final Color backgroundInput;
  final Color textDefault;

  // 私有的构造函数
  const AppColorsTheme._internal({
    required this.backgroundDefault,
    required this.backgroundInput,
    required this.textDefault,
  });

  // 浅色主题工厂方法
  factory AppColorsTheme.light() {
    return const AppColorsTheme._internal(
        backgroundDefault: _colorB0B0B0,
        backgroundInput: _colorEFEFEF,
        textDefault: _color333333);
  }

  // 暗色主题工厂方法
  factory AppColorsTheme.dark() {
    return const AppColorsTheme._internal(
        backgroundDefault: _color6C6C6C,
        backgroundInput: _color7D7D7D,
        textDefault: _color676767);
  }

  @override
  ThemeExtension<AppColorsTheme> copyWith({bool? lightMode}) {
    if (lightMode == null || lightMode == true) {
      return AppColorsTheme.light();
    }
    return AppColorsTheme.dark();
  }

  @override
  ThemeExtension<AppColorsTheme> lerp(
          covariant ThemeExtension<AppColorsTheme>? other, double t) =>
      this;
}
```
这里将`color name`  与页面中实际使用的颜色变量名称分开，因为很多时候 `color name` 在不同的主题模式下是不一样的，虽然只提供了 `light` 和 `dark`，当然我还可以添加更多其它的主题颜色。

### `Text Theme`

创建文本样式主题，命名为 `AppTextsTheme`，同样继承自 `ThemeExtension`。

```Dart
import 'package:flutter/material.dart';

class AppTextsTheme extends ThemeExtension<AppTextsTheme> {
  static const _baseFamily = "Roboto";

  final TextStyle labelH1;
  final TextStyle labelH2;
  final TextStyle labelTextDefault;

  const AppTextsTheme._internal(
      {required this.labelH1,
      required this.labelH2,
      required this.labelTextDefault});

  factory AppTextsTheme.main() => const AppTextsTheme._internal(
      labelH1: TextStyle(
        fontFamily: _baseFamily,
        fontWeight: FontWeight.w400,
        fontSize: 18,
        height: 1.4,
      ),
      labelH2: TextStyle(
        fontFamily: _baseFamily,
        fontWeight: FontWeight.w300,
        fontSize: 16,
        height: 1.4,
      ),
      labelTextDefault: TextStyle(
        fontFamily: _baseFamily,
        fontWeight: FontWeight.w400,
        fontSize: 16,
        height: 1.2,
      ));

  @override
  ThemeExtension<AppTextsTheme> copyWith() {
    return AppTextsTheme._internal(
      labelH1: labelH1,
      labelH2: labelH2,
      labelTextDefault: labelTextDefault,
    );
  }

  @override
  ThemeExtension<AppTextsTheme> lerp(
          covariant ThemeExtension<AppTextsTheme>? other, double t) =>
      this;
}
```
这里的字体等文本样式需要和 `UI` 设计人员沟通好，而且最好命名也和他们设计稿的名称保持一致，这样在后期重复使用的时候能最大的降低沟通成本。

### `Dimension Theme`

`AppDimensionsTheme` 主要存放项目中的尺寸相关的主题数据，例如间距、大小、边框宽度等。这些尺寸数据通常用于保持应用程序中的视觉一致性和布局稳定性。

```Dart
import 'package:flutter/material.dart';

class AppDimensionsTheme extends ThemeExtension<AppDimensionsTheme> {
  final double radiusCommitButton;
  final EdgeInsets paddingOrderList;

  const AppDimensionsTheme._internal({
    required this.radiusCommitButton,
    required this.paddingOrderList,
  });

  factory AppDimensionsTheme.main() => const AppDimensionsTheme._internal(
        radiusCommitButton: 8,
        paddingOrderList: EdgeInsets.symmetric(horizontal: 12, vertical: 10),
      );

  @override
  ThemeExtension<AppDimensionsTheme> copyWith() {
    return AppDimensionsTheme._internal(
      radiusCommitButton: radiusCommitButton,
      paddingOrderList: paddingOrderList,
    );
  }

  @override
  ThemeExtension<AppDimensionsTheme> lerp(
          covariant ThemeExtension<AppDimensionsTheme>? other, double t) =>
      this;
}
```
这里一般是放一些比较通用的尺寸，而不是将所有的和尺寸相关的都放在这里。你可能会问，尺寸都写死在这里，那如何做响应式 `UI` 呢？

### 响应式 `UI`

为 `FlutterView` 创建一个 `extension`，命名为 `FlutterViewExtension`。
```Dart
import 'package:flutter/gestures.dart';

extension FlutterViewExtension on FlutterView {
  static const double responsive360 = 360;
  static const double responsive480 = 480;
  static const double responsive600 = 600;
  static const double responsive800 = 800;
  static const double responsive900 = 900;
  static const double responsive1200 = 1200;

  double get logicalWidth => physicalSize.width / devicePixelRatio;

  double get logicalHeight => physicalSize.height / devicePixelRatio;

  double get logicalWidthSA =>
      (physicalSize.width - padding.left - padding.right) / devicePixelRatio;

  double get logicalHeightSA =>
      (physicalSize.height - padding.top - padding.bottom) / devicePixelRatio;

  bool get isSmallSmartphone {
    if (logicalWidthSA < logicalHeightSA) {
      return (logicalWidthSA <= responsive360 ||
          logicalHeightSA <= responsive600);
    } else {
      return (logicalWidthSA <= responsive600 ||
          logicalHeightSA <= responsive360);
    }
  }

  bool get isRegularSmartphoneOrLess {
    if (logicalWidthSA < logicalHeightSA) {
      return (logicalWidthSA <= responsive480 ||
          logicalHeightSA <= responsive800);
    } else {
      return (logicalWidthSA <= responsive800 ||
          logicalHeightSA <= responsive480);
    }
  }

  bool get isSmallTabletOrLess {
    if (logicalWidthSA < logicalHeightSA) {
      return (logicalWidthSA <= responsive600 ||
          logicalHeightSA <= responsive900);
    } else {
      return (logicalWidthSA <= responsive900 ||
          logicalHeightSA <= responsive600);
    }
  }

  bool get isRegularTabletOrLess {
    if (logicalWidthSA < logicalHeightSA) {
      return (logicalWidthSA <= responsive800 ||
          logicalHeightSA <= responsive1200);
    } else {
      return (logicalWidthSA <= responsive1200 ||
          logicalHeightSA <= responsive800);
    }
  }
}

```

在 `AppDimensionsTheme` 中使用，只需要在这里加上判断就可以了。
```Dart
factory AppDimensionsTheme.main(FlutterView flutterView) =>
  AppDimensionsTheme._internal(
    radiusCommitButton: flutterView.isSmallSmartphone ? 8 : 16,
    paddingOrderList:
        const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
  );
```


### 如何使用 `Theme Extension`

第一步：在 `main.dart` 文件，将以下代码放在程序入口 `Widget` 的 `MaterialApp` 下面。

```Dart
MaterialApp(
  title: 'Flutter技术实践',
  theme: Theme.of(context).copyWith(
    extensions: [
      AppTextsTheme.main(),
      AppColorsTheme.light(),
      AppDimensionsTheme.main(View.of(context)),
    ],
  )
)
```
第二步：为 `ThemeData` 创建一个 `extension`，目的是简化了调用代码。

```Dart
import 'package:flutter/material.dart';
import 'package:flutter_todo/resources/app_colors_theme.dart';
import 'package:flutter_todo/resources/app_dimensions_theme.dart';
import 'package:flutter_todo/resources/app_texts_theme.dart';

extension ThemeDataExtension on ThemeData {
  AppDimensionsTheme get appDimensions => extension<AppDimensionsTheme>()!;

  AppColorsTheme get appColors => extension<AppColorsTheme>()!;

  AppTextsTheme get appTexts => extension<AppTextsTheme>()!;
}
```

第三步：开始调用，这里显示文本样式为 `labelTextDefault`，颜色为 `textDefault`。
```Dart
Text(
  "Flutter",
  style: Theme.of(context).appTexts.labelTextDefault.copyWith(
    color: Theme.of(context).appColors.textDefault,
  ),
),
```

## 分层架构中去管理设计系统

![架构分层](https://s2.loli.net/2024/04/21/7oCVc96UkvN2bI5.png)

从上面的分层设计架构图可以到，把与 `Theme Extension` 相关的划分到 `resources` 的组件包（在上图红框内），`resources`  组件包的目录结构如下：

![resources 组件包目录结构](https://s2.loli.net/2024/04/21/R39magV4F1pHLAW.png)

而`resources` 的组件包在整个架构中是作为基础组件包，不需要依赖任何其它组件包，这样在使用 `ThemeExtension` 时可以确保整个应用程序中使用的主题样式保持一致，从而提高用户体验和视觉一致性。

## 小结
本篇介绍了 `Flutter` 大型项目分层架构中的UI设计系统实现，主要是原子级别的，如 `color`、`font`、`padding`、`radius` 等等设计系统的基础，下一篇来介绍设计系统中分子级别和细胞级别，感谢您的阅读，更多该系列干货文章请关注我关注号：**Flutter技术实践**，记得关注加点赞哦！

![Flutter技术实践](https://s2.loli.net/2022/11/09/fNBn2gWw8tVazkr.jpg)