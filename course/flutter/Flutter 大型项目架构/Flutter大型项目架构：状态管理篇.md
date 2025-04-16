---
title: Flutter 大型项目架构：状态管理篇
date: 2023-08-09 16:59:26
tags:
---

最近对手头上的项目进行重构，总结了有以下几个痛点：
1. 状态管理混乱
虽然用了 `provider` 来做状态管理，但一些代码如：异步请求、事件响应等还是会掺杂在UI页面的代码里面，一旦页面的各种 `Widget` 多了起来之后，显得非常严重，而且对业务逻辑的测试也不方便，多个组件可能需要共享相同的数据或状态，需要在每个组件中分别创建 `Provider` 实例，容易导致代码冗余，如果只需要更新页面的部分 `Widget` 使用`Provider` 还会导致嵌套过深，也可能导致性能问题、状态不一致以及难以追踪的错误。
2. 代码分层混乱，所有的代码都在一个工程里面，包括UI页面、`ViewModel`、`Util`、网络请求和通用的 `Widget` 等，模块划分不清晰，模块依赖和调用混乱，而且各个模块没发单独开发和测试，没有明确的分层可能导致业务逻辑和 UI 代码混杂在一起，而且多个项目之间很难做到代码复用。
3. 业务迭代导致API接口变动频繁，API接口返回的字段一旦出现类型解析问题就会导致出现崩溃，而目前负责 `Json` 映射的数据模型 `Model` 类在很多的 UI `Widget` 和 `Bloc` 中有使用到，修复这个问题就会导致大量依赖于此类的改动。而为了安全起见，数据模型类中的所有字段必须始终是这样的：`int？`、`string？`等，但是，这又可能会导致由于 `NullPoInterException` 而出现应用程序崩溃的风险。
4. 路由管理问题，`fluro`作为路由管理必须依赖于 `context`，如果在逻辑层需要处理页面跳转的话就会比较麻烦。而且参数传递只能作为转成字符串放在 `path` 中，然后再在注册路由的 `Handler` 中将字段解析出或者转成 `Model`，使用起来多少有些不方便。

接下来系列文章来聊一聊怎解决上面的痛点，本篇文章先说一下状态管理方法，现有的项目需要一个能很好地应付大型项目的状态管理机制。

<!-- more -->

## 什么是状态管理

Flutter 状态管理是指在 Flutter 应用中有效地管理应用的数据和状态，以确保用户界面（UI）与数据之间的一致性和交互性。在复杂的应用中，数据通常会在不同的部分之间流动和变化，而状态管理的目标是帮助开发者更好地组织、更新和共享这些数据。

下面是使用比较多的状态管理方法及优缺点：

![状态管理方法优缺点](https://s2.loli.net/2023/08/07/5LW24hRuAZ1c9p3.png)

上面可以看出如果是小型项目用 `setState()` 或 `InheritedWidget`，原生自带，简单快捷。大型项目可以选择更复杂的方法，如 `Provider`、`BLoC` 或 `Redux`。特别是对于复杂的应用逻辑，`BLoC` 和 `Redux` 提供了更强大的状态管理和业务分离，将逻辑业务从UI层中剥离，方便维护和拓展功能，但同时学习的复杂度也高一些。在项目初期选择合适的状态管理方法还是很重要的，提升效率的同时也避免了后面频繁更换引起大量代码的变更。

本系列文章主要是针对大型项目的架构来展开的，所以选择了 `BLoC` 作为状态管理方法，那为什么是 `BLoC` ?

## Why Bloc?

> Bloc makes it easy to separate presentation from business logic, making your code fast, easy to test, and reusable.

引用了 [bloclibrary.dev](https://bloclibrary.dev/#/whybloc) 中一句话：`Bloc` 可以轻松地将展示层与业务逻辑分离，使您的代码快速、易于测试且可重用。
![](https://fastly.jsdelivr.net/gh/filess/img19@main/2023/08/08/1691458663009-b32a0c28-5613-409b-85c5-45cbb573a236.png)

在 `Flutter BLoC` 架构中，应用的业务逻辑被组织为一个个的 `BLoC`。每个 `BLoC` 是一个独立的单元，一情况下对应一个页面（Page），负责管理这个页面特定的状态和逻辑，如网络请求、请求返回的数据解析处理、点击事件等等。`BLoC` 通过 `Streams`（流）和 `Sink`（数据源）与 UI 组件通信。这种分离的架构使得 UI 组件只需关注显示数据，而将数据获取、处理和变化交给了相应的 `BLoC`。

`Bloc` 主要由以下几个部分组成：

`State`： `Bloc` 包含一个状态对象，它存储了当前的应用状态。这个状态可以是任何数据类型，从简单的布尔值到复杂的对象都可以。

````dart
class HomeState  {
  bool isShimmerLoading;
  int selectedIndex;
  dynamic responseData;
}
````

`Event`： 事件是触发状态变化的操作或用户动作。当用户与应用交互时，事件会被触发，然后 BLoC 根据事件来决定如何改变状态。

````dart
abstract class HomeEvent {
  const HomeEvent();
}

class HomePageInitiated extends HomeEvent  {
  const HomePageInitiated(this.status);
  final AuthenticationStatus status;
}

class HomePageRefreshed extends HomeEvent {
  const HomePageRefreshed(this.completer);
  Completer<void> completer;
}
````

`Bloc`： `Bloc` 中包含了实际的业务逻辑。根据接收到的事件，`Bloc` 可以对状态进行更新、计算、处理异步操作等。

```dart
class HomeBloc extends Bloc<HomeEvent, HomeState> {
  HomeBloc() : super(HomeState()) {
    on<HomePageInitiated>(
      _onHomePageInitiated
    );

    on<HomePageRefreshed>(
      _onHomePageRefreshed
    );
  }
  
  FutureOr<void> _onHomePageInitiated(
      HomePageInitiated event, Emitter<HomeState> emit)  {
      // 发出一个新状态，通知页面更新。
      emit(state.copyWith(isShimmerLoading: false));
  }
    
  FutureOr<void> _onHomePageRefreshed(
      HomePageRefreshed event, Emitter<HomeState> emit)  {
  }
 }
```

UI Page：`Bloc` 使用 `Streams` 将状态传递给 UI 组件，UI 组件可以订阅 `Bloc` 的 `Stream` 来监听状态变化，同时通过 `Sink` 向 `Bloc` 发送事件，触发状态变化。

```dart
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Home')),
      body: Padding(
        padding: const EdgeInsets.all(12),
        child: BlocProvider(
            create: (context) {
              return HomeBloc();
            },
            child: BlocBuilder<HomeBloc, HomeState>(
              buildWhen: (previous, current) => previous.isShimmerLoading != current.isShimmerLoading,
              builder: (context, state) {
                return GestureDetector(
                  onTap: () {
                    bloc.add(const HomePageInitiated());
                  },
                  child: ......,
                );
              },
            )
        ),
      ),
    );
  }
```

这样做的优势也很明显：

- 业务逻辑分离： `Bloc` 架构将业务逻辑从 UI 中分离，使代码更具可读性和可维护性，同时便于单元测试。
- 可测试性： 由于 `Bloc` 的状态和逻辑都是独立的，可以轻松地进行单元测试，确保代码质量。
- 状态可预测： 使用 `Streams` 将状态传递给 UI 组件，确保状态的一致性和可预测性。


## Bloc 实际应用
接下来实现`Bloc`中发送网络请求，获取数据后再刷新页面，实际应用中会导入`freezed`、`injectable`和`get_it`等依赖库，其中 [freezed](https://github.com/rrousselGit/freezed) 是数据体代码生成器，[injectable](https://github.com/Milad-Akarie/injectable) 搭配 [get_it](https://github.com/fluttercommunity/get_it) 解决依赖注入的问题。还有其它依赖如下：

```yaml
name: app
description: A new Flutter project.
publish_to: 'none' # Remove this line if you wish to publish to pub.dev
version: 1.0.0+1

environment:
  sdk: ">=2.15.0 <3.0.0"

dependency_overrides:
  analyzer: 5.2.0
  build_resolvers: 2.1.0
  dart_style: 2.2.4
  ffi: ^1.2.0

dependencies:
  flutter_localizations:
    sdk: flutter
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.2
  
  # 以下这几个package后面会有文章单独来说
  widgets:
    path: ../widgets
  shared:
    path: ../shared
  domain:
    path: ../domain
  resources:
    path: ../resources
  initializer:
    path: ../initializer
    
  injectable:
  get_it:
  flutter_bloc: 8.0.1
  freezed_annotation: 2.2.0
  flutter_screenutil: 5.5.3+2
  auto_route: 5.0.4
  loading_animation_widget: ^1.2.0+4
  pull_to_refresh: ^2.0.0
  url_launcher:

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^1.0.0
  injectable_generator: ^2.1.4
  build_runner: 2.3.3
  freezed: 2.3.2
  auto_route_generator: 5.0.3
  flutter_gen_runner: 4.1.6

flutter_gen:
  output: lib/resource/generated
  line_lenght: 160
  null_safety: true

  integrations:
    flutter_svg: true

  assets:
    enabled: true

  fonts:
    enabled: true

flutter:
  uses-material-design: true
  generate: false
  assets:
    - assets/images/
```
以 Home 页面为例，文件创建好后目录结构大概是这样的：
![image.png](https://s2.loli.net/2023/08/09/QGWvPlkhiMcAw3q.png)

其中 `home_event.freezed.dart` 和 `home_state.freezed.dart` 需要手动创建好，内容是添加 `@freezed` 注解后自动生成的，不需要手动更改内容。

#### home_event.dart

```dart
import 'dart:async';
import 'package:freezed_annotation/freezed_annotation.dart';
import '../../../base/bloc/base_bloc_event.dart';
part 'home_event.freezed.dart';

abstract class HomeEvent extends BaseBlocEvent {
  const HomeEvent();
}

@freezed
class HomePageInitiated extends HomeEvent with _$HomePageInitiated {
  const factory HomePageInitiated() = _HomePageInitiated;
}
```

#### home_state.dart

```dart
import 'package:domain/domain.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:shared/shared.dart';
import '../../../base/bloc/base_bloc_state.dart';
part 'home_state.freezed.dart';

@freezed
class HomeState extends BaseBlocState with _$HomeState {
  factory HomeState({
    @Default([]) List<PlayListItemData> playList,
  }) = _HomeState;
}
```

#### home_bloc.dart

```dart
import 'dart:async';
import 'package:app/base/bloc/base_bloc.dart';
import 'package:domain/domain.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:injectable/injectable.dart';
import 'package:app/ui/home/bloc/home_event.dart';
import 'package:app/ui/home/bloc/home_state.dart';
import 'package:shared/util/log_utils.dart';

@Injectable()
class HomeBloc extends BaseBloc<HomeEvent, HomeState> {
  HomeBloc(this._repository) : super(HomeState()) {
    on<HomePageInitiated>(
      _onHomePageInitiated,
      transformer: log(),
    );

    on<UserLoadMore>(
      _onUserLoadMore,
      transformer: log(),
    );

    on<HomePageRefreshed>(
      _onHomePageRefreshed,
      transformer: log(),
    );
  }

  final Repository _repository;
  
  FutureOr<void> _onHomePageInitiated(
      HomePageInitiated event, Emitter<HomeState> emit) async {
    try {
      List<PlayListItemData> playList = await _repository.playlist("");
      // 获取数据后触发页面更新
      emit(state.copyWith(playList: playList));
    } catch (err) {
      Log.e(err);
    }
  }
}
```

`Repository` 是数据请求的接口，其实现类是`RepositoryImpl`，通过注解`@LazySingleton(as: Repository)` 将实现类`RepositoryImpl` 注入到 `di` 中，这一块儿将在网络层的重构会说到。


`HomePage` 的 `_HomePageState` 继承自 `BasePageState`，而 `BasePageState`继承`BasePageStateDelegate` 并在 `BasePageStateDelegate` 注册 `Bloc`

#### base_page_state.dart

```dart
abstract class BasePageState<T extends StatefulWidget, B extends BaseBloc>
    extends BasePageStateDelegate<T, B> with LogMixin {}

abstract class BasePageStateDelegate<T extends StatefulWidget,
    B extends BaseBloc> extends State<T> implements ExceptionHandlerListener {
  late final AppNavigator navigator = GetIt.instance.get<AppNavigator>();

  late final B bloc = GetIt.instance.get<B>()
    ..navigator = navigator;

  @override
  Widget build(BuildContext context) {
    return Provider(
      create: (context) {},
      child: MultiBlocProvider(
        providers: [
          BlocProvider(create: (_) => bloc),
        ],
        child: buildPageListeners(
            child: isAppWidget
                ? buildPage(context)
                : Stack(
                    children: [
                      buildPage(context),
                      BlocBuilder<CommonBloc, CommonState>(
                        buildWhen: (previous, current) =>
                            previous.isLoading != current.isLoading,
                        builder: (context, state) => Visibility(
                          visible: state.isLoading,
                          child: buildPageLoading(),
                        ),
                      ),
                    ],
                  ),
          )
      ),
    );
  }

  Widget buildPageListeners({required Widget child}) => child;

  Widget buildPageLoading() => const Center(
        child: CircularProgressIndicator(
          color: Color(0xFF666666),
          strokeWidth: 2,
        ),
      );

  Widget buildPage(BuildContext context);

  @override
  void dispose() {
    super.dispose();
  }
}
```



#### home_page.dart

```dart
import 'package:app/ui/home/bloc/home_state.dart';
import 'package:flutter/material.dart';
import 'package:app/base/base_page_state.dart';
import 'package:app/common_view/common_scaffold.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'bloc/home_bloc.dart';
import 'bloc/home_event.dart';

class HomePage extends StatefulWidget {
  const HomePage({Key? key}) : super(key: key);

  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends BasePageState<HomePage, HomeBloc> {
  @override
  initState() {
    super.initState();
    // 向 bloc 添加 HomePageInitiated event
    bloc.add(const HomePageInitiated());
  }

  @override
  Widget buildPage(BuildContext context) {
    return CommonScaffold(
        backgroundColor: Colors.white,
        hideKeyboardWhenTouchOutside: true,
        body: BlocBuilder<HomeBloc, HomeState>(
          buildWhen: (previous, current) =>
              previous.playList != current.playList,
          builder: (ctx, state) {
            return ListView.separated(
                itemBuilder: (BuildContext context, int index) {
                  return SizedBox(
                    height: 120,
                    child: Text(state.playList[index].name ?? ""),
                  );
                },
                separatorBuilder: (BuildContext context, int index) {
                  return Container();
                },
                itemCount: state.playList.length);
          },
        ));
  }
}
```

在 `buildWhen` 中对比前后数据不一样的时候就会触发更新 `ListView` 的显示，而且只需要判断监听 `playList` 这一个字段的数据变化，如果需要监听其它字段来更新其它的 `Widget` 时也互相不干扰，，整个没有多余的逻辑处理，UI 层干净清爽多了。以事件为驱动，事件一旦被触发，然后 `BLoC` 根据事件来决定如何改变状态，好处是这里的状态和逻辑都是独立的，可以轻松地进行单元测试来确保代码质量。



欢迎关注我的公众号“**flutter技术实践**”，原创技术文章第一时间推送。

<center>
    <img src="https://s2.loli.net/2022/12/12/EWvyFX2LgiaeZ3d.jpg" style="width: 100px;">
</center>
