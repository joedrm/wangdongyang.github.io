---
title: Flutter大型项目架构：路由管理篇
date: 2024-05-18 10:33:42
tags:
---

![](https://s2.loli.net/2024/05/18/LpIQhSkEji1DgR3.png)

在本系列的依赖管理篇讲到了以路由依赖为例子来介绍如何做依赖设计的，具体操作就是将抽象类 `AppNavigator` 和实现类 `AppNavigatorImpl` 通过依赖注入的方式联系起来，而在使用的时候调用 `AppNavigator`  ，不再关心具体的实现逻辑，这种设计在做组件分层和处理多个组件间的依赖关系的时候显得尤为重要，也很好的诠释了软件架构设计中的 **依赖于抽象而不是具体的实现**。但是关于 `Flutter` 中路由管理知识以及在大型项目中如何做路由设计很少有介绍，本篇就来说一说路由管理在大型的项目的实践。

<!-- more -->

## `Flutter`原生路由

记得刚开始开发 `Flutter` 应用的时候，跳转页面直接使用 `Navigator.of(context).push` ，返回 `Navigator.pop`，简单快捷，传参也方便，还能定制页面跳转的动画，从 `iOS` 原生开发转过来开发 `Flutter` 也能很快适应。随着应用的功能越来越多，逻辑也变得复杂，各种页面跳转及传参，发现了一些原生的路由（`Navigator` 和 `Route`）的弊端。

对于更复杂的导航需求，比如深层嵌套的导航、条件导航、全局导航钩子等，原生路由的能力不足，比如一个常见多层次的开发场景，有三个页面：`HomeScreen`、`CategoryScreen` 和 `ItemScreen`。用户需要通过 `CategoryScreen` 导航到 `ItemScreen`，并从 `ItemScreen` 返回到 `CategoryScreen` 或 `HomeScreen`。

```dart
// 配置路由表
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      initialRoute: '/',
      routes: {
        '/': (context) => HomeScreen(),
        '/category': (context) => CategoryScreen(),
        '/item': (context) => ItemScreen(),
      },
    );
  }
}
//  HomeScreen -> CategoryScreen
Navigator.pushNamed(context, '/category');

// CategoryScreen -> ItemScreen
Navigator.pushNamed(context, '/item');

// ItemScreen -> HomeScreen
Navigator.popUntil(context, ModalRoute.withName('/'));
```
上面例子可以看出来，深层嵌套导航时候需要手动去管理返回路径和路由堆栈，这还没有加上各种复杂的参数传递，在复杂的而应用中这样使用太麻烦了，还容易出错。当然如果你觉得上面这种写法还能接受，那么原生路由应用在处理 `tab` 页面中的嵌套路由的时候又会是什么情况呢？

那好办啊，给每个 `Tab` 页面中使用一个独立的 `Navigator` 组件来管理该 `Tab` 内部的路由，这样可以使每个 `Tab` 页面具有独立的路由堆栈而互不影响，不就解决了嘛。嗯，好像是可以哈，这时候产品的需求来了：某个 `tab` 的子页面回到其它的 `tab` 首页（如下图所示），该怎么实现跳转呢？答案是没有很好的办法，因为每个 `tab` 中的 `Navigator` 是相互独立的，各自管理自己的路由堆栈。除非是通过全局状态管理来记录 `tab` 页面的状态，但是这样做复杂度又上升了。

![tab_stucture.png](https://s2.loli.net/2024/05/17/1fmULcvRJndYST5.png)

条件导航、全局导航钩子也很理解，比如说对某些页面的访问进行权限控制，用户在未登录时访问主页需要重定向到登录页面，可以使用 `NavigatorObserver` 。

```dart
class AuthGuard extends NavigatorObserver {
  @override
  void didPush(Route<dynamic> route, Route<dynamic>? previousRoute) {
    super.didPush(route, previousRoute);

    if (route.settings.name == '/protected' && !isLoggedIn) {
      // 如果用户未登录，则重定向到登录页面
      WidgetsBinding.instance.addPostFrameCallback((_) {
        route.navigator?.pushReplacementNamed('/login');
      });
    }
  }

  bool get isLoggedIn => false; // 假设这是用户登录状态
}
```
使用 `AuthGuard`
```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      initialRoute: '/',
      routes: {
        '/': (context) => HomeScreen(),
        '/protected': (context) => ProtectedScreen(),
        '/login': (context) => LoginScreen(),
      },
      navigatorObservers: [AuthGuard()],
    );
  }
}
```

虽然可以使用 `NavigatorObserver` 实现类似需求，但使用起来还是不够方便简洁，而且当我们需要对某个页面添加 `Observer` 的时候，特别是需要对某些页面的访问进行特殊权限控制的时候，使用原生的  `NavigatorObserver`  更加麻烦了，下面的例子在 `HomeScreen` 跳转到 `DetailsScreen` 之间添加 `Observer` 。

```dart
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (context) => ObserverPage(
      observer: MyPageObserver(),
      child: DetailsScreen(),
    ),
  ),
);
```

包装器 `ObserverPage` 类 和 `MyPageObserver` 的实现

```dart
class ObserverPage extends StatelessWidget {
  final Widget child;
  final NavigatorObserver observer;

  ObserverPage({required this.child, required this.observer});

  @override
  Widget build(BuildContext context) {
    return Navigator(
      observers: [observer],
      onGenerateRoute: (settings) {
        return MaterialPageRoute(
          builder: (context) => child,
        );
      },
    );
  }
}

class MyPageObserver extends NavigatorObserver {
  @override
  void didPush(Route route, Route? previousRoute) {
    super.didPush(route, previousRoute);
    // 处理特殊的业务逻辑
    if ("判断条件") {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        route.navigator?.pushReplacementNamed('/xx');
      });
    }
  }
}
```
这种情况下使用自定义的 `NavigatorObserver` 太过麻烦了，有没有？一个小小的需求竟然要加这么代码，同样的问题在参数传递的过程中也有。

原生路由在完成规模不大的项目时是能应对的，但是对于页面较多，跳转逻辑也比较复杂的应用它的弊端就会凸显出来了，那么大型项目是如何处理上面的问题呢？又是如何做路由管理的？

## `AppNavigator` 的实现

为什么是 `auto_router` 而不是 `go_router` 呢？个人觉得因人而异吧，`auto_router` 刚好能我的使用需求，通过注解的方式配置路由，需要编写的代码量较少，而且支持嵌套路由、参数传递、路由守卫等高级功能，也可以很方便地与 `Provider`、`Bloc` 等状态管理库集成，上面介绍使用原生路由带来的问题在 `auto_router` 都有相应的解决方案，具体的使用大家可以看看它的文档。本篇主要讲一讲在大型的 `Flutter` 中如何使用和管理路由的。

在做分层设计的时候，往往将抽象和具体实现分开，路由管理中就是抽象类 `AppNavigator` 放在 `domian` 组件包中，实现类 `AppNavigatorImpl` 直接放在了主工程中，通过依赖注入来关联彼此，下面来详细介绍一下简化版的实现类 `AppNavigatorImpl` 。

```dart
import 'package:app/navigation/routes/app_router.dart';
import 'package:auto_route/auto_route.dart';
import 'package:domain/domain.dart';
import 'package:flutter/material.dart' as m;
import 'package:injectable/injectable.dart';
import 'package:app/navigation/base/base_popup_info_mapper.dart';
import 'package:app/navigation/base/base_route_info_mapper.dart';
import 'package:shared/shared.dart';

@LazySingleton(as: AppNavigator)
class AppNavigatorImpl extends AppNavigator with LogMixin {
  AppNavigatorImpl(
    this._appRouter,
    this._appRouteInfoMapper,
  );

  final tabRoutes = const [
    OrderTab(),
    HomeTab(),
    MyPageTab(),
  ];
  
  TabsRouter? tabsRouter;

  final AppRouter _appRouter;
  final BaseRouteInfoMapper _appRouteInfoMapper;
  
  StackRouter? get _currentTabRouter => tabsRouter?.stackRouterOfIndex(currentBottomTab);
  StackRouter get _currentTabRouterOrRootRouter => _currentTabRouter ?? _appRouter;

  @override
  Future<T?> push<T extends Object?>(AppRouteInfo appRouteInfo) {
    return _appRouter.push<T>(_appRouteInfoMapper.map(appRouteInfo));
  }

  @override
  Future<bool> pop<T extends Object?>({T? result, bool useRootNavigator = false}) {
    return useRootNavigator
        ? _appRouter.pop<T>(result)
        : _currentTabRouterOrRootRouter.pop<T>(result);
  }
  
  @override
  int get currentBottomTab {
    if (tabsRouter == null) {
      throw 'Not found any TabRouter';
    }

    return tabsRouter?.activeIndex ?? 0;
  }
  
  @override
  void popUntilRootOfCurrentBottomTab() {
    if (tabsRouter == null) {
      throw 'Not found any TabRouter';
    }

    if (_currentTabRouter?.canPop() == true) {
      _currentTabRouter?.popUntilRoot();
    }
  }

  @override
  void navigateToBottomTab(int index, {bool notify = true}) {
    if (tabsRouter == null) {
      throw 'Not found any TabRouter';
    }
    tabsRouter?.setActiveIndex(index, notify: notify);
  }
  
  // 还有其它实现，这里先省略...
}
```
`AppNavigatorImpl` 构造方法中需注入三个实例，分别是 `AppRouter` 用于管理应用导航的核心类，是全局路由管理单例类，项目中用到所有的路由配置都集中在 `AppRouter`，通过使用 `auto_route` 包为 `AppRouter` 提供了一种结构化和类型安全的方式来定义和处理应用中的路由，下面是 `AppRouter` 配置代码：

```dart
import 'package:auto_route/auto_route.dart';
import 'package:injectable/injectable.dart';

part 'app_router.gr.dart';

@AutoRouterConfig(
  replaceInRouteName: 'Page,Route',
)
@LazySingleton()
class AppRouter extends _$AppRouter {
  @override
  RouteType get defaultRouteType => const RouteType.adaptive();

  @override
  List<AutoRoute> get routes => [
    	AutoRoute(page: LoginRoute.page),
        AutoRoute(page: MainRoute.page, initial: true, children: [
            AutoRoute(
              page: HomeTab.page,
              maintainState: true,
              children: [
                AutoRoute(page: HomeRoute.page, initial: true),
              ],
            ),
            AutoRoute(
              page: OrderTab.page,
              maintainState: true,
              children: [
                AutoRoute(page: OrderRoute.page, initial: true),
              ],
            ),
            AutoRoute(
              page: MyPageTab.page,
              maintainState: true,
              children: [
                AutoRoute(page: MyPageRoute.page, initial: true),
              ],
            ),
        ]),
      ];
}

@RoutePage(name: 'HomeTab')
class HomeTabPage extends AutoRouter {
  const HomeTabPage({super.key});
}

@RoutePage(name: 'OrderTab')
class OrderTabPage extends AutoRouter {
  const OrderTabPage({super.key});
}

@RoutePage(name: 'MyPageTab')
class MyPageTabPage extends AutoRouter {
  const MyPageTabPage({super.key});
}
```
在 `AppRouter` 配置了 `LoginPage`、`MainTabPage` 主页及下面的3个 `tab` 页面，在类 `AppNavigatorImpl`  使用 `_appRouter` 实例来实现各种页面的跳转，如 `push`、`pop`  等操作， 

而 `tabsRouter`  可以切换当前显示的 `tab` 页，如 `navigateToBottomTab` 函数，但是好像没有看到在哪里给 `tabsRouter` 赋值啊，其实 `tabsRouter` 是在 `MainPage` 中实现的。

```dart
@override
Widget buildPage(BuildContext context) {
	// 这里的 AutoTabsScaffold 也是 auto_router 提供用来创建 AutoTabsRouter 的
  return AutoTabsScaffold(
    routes: (navigator as AppNavigatorImpl).tabRoutes,
    bottomNavigationBuilder: (_, tabsRouter) {
    	// 此处给 `tabsRouter` 赋值
      (navigator as AppNavigatorImpl).tabsRouter = tabsRouter;
      return ...;
    },
  );
}
```

通过 `tabsRouter`  获取当前的正在显示的 `_currentTabRouter` 可将子页面回退到栈底的 `tab` 页，如 `popUntilRootOfCurrentBottomTab`  操作。

那 `AppNavigatorImpl`  注入的这个 `BaseRouteInfoMapper` 又是干什么用的呢？`BaseRouteInfoMapper` 是一个抽象类，其实现类是 `AppRouteInfoMapper` ，也是通过依赖注入的方式将两者绑定。

```dart
abstract class BaseRouteInfoMapper {
  PageRouteInfo map(AppRouteInfo appRouteInfo);
}

@LazySingleton(as: BaseRouteInfoMapper)
class AppRouteInfoMapper extends BaseRouteInfoMapper {
  @override
  PageRouteInfo map(AppRouteInfo appRouteInfo) {
	 // 
    return appRouteInfo.when(
      login: () => const LoginRoute(),
  }
}
```
重写父类的 `map` 方法中会根据 `AppRouteInfo` 不同的工厂方法返回具体的 `PageRouteInfo` 对象，这里的是 `LoginRoute`，而 `LoginRoute` 是使用 `auto_router` 生成器来自动生成的，指向具体的路由。

```dart
import 'package:freezed_annotation/freezed_annotation.dart';
part 'app_route_info.freezed.dart';

@freezed
class AppRouteInfo with _$AppRouteInfo {
  const factory AppRouteInfo.login() = _Login;
}
```
上面代码的作用是定义了一个不可变类 `AppRouteInfo`，并使用 `@freezed` 注解。这个类有一个工厂构造函数 `login`，当我们调用 `AppRouteInfo.login()` 时，它会创建一个 `_Login` 类型的实例，当然还可以定义更多的工厂构造函数并隐藏具体的实现类。所以在 `AppRouteInfoMapper` 的 `map` 函数中使用 `when` 可工具不同工厂方法返回不同类型的 `PageRouteInfo` 实例。

那么再回到 `AppNavigatorImpl` 看看 `push` 方法的实现。

```dart
  @override
  Future<T?> push<T extends Object?>(AppRouteInfo appRouteInfo) {
    return _appRouter.push<T>(_appRouteInfoMapper.map(appRouteInfo));
  }
```
这里的 `appRouteInfo` 就是传 `AppRouteInfo.login()` 来实现路由跳转，具体的页面调用就一句代码 `navigator.push(const AppRouteInfo.login());` 

上面的 `navigator` 是在哪里实例化的呢？在这里是将 `AppNavigator` 的实例挂在自定义的 `State` 基类中，这样方便在所有的子页面使用 `navigator` 做页面跳转。

```dart
// 自定义的 State 基类
abstract class BasePageState<T extends StatefulWidget,
    B extends BaseBloc> extends State<T> {
  // 导航器对象
  late final AppNavigator navigator = GetIt.instance.get<AppNavigator>();
}
```

## `bloc` 中使用 `navigator` 跳转页面

使用 `bloc` 进行状态管理时，当某些业务逻辑处理完成时，需要根据结果进行页面跳转。例如，用户登录成功后跳转到主页，或者表单提交成功后跳转到成功页面。这种情况下，跳转逻辑通常位于 `Bloc` 内，以保业务逻辑和跳转逻辑的统一。那么这种情况改如何实现呢？

首先我们可以在 `bloc` 的基类中声明一个 `navigator` ，`BaseBloc` 的实现如下：

```dart
abstract class BaseBloc<E extends BaseBlocEvent, S extends BaseBlocState>
    extends BaseBlocDelegate<E, S> {
  BaseBloc(S initialState) : super(initialState);
}

abstract class BaseBlocDelegate<E extends BaseBlocEvent,
    S extends BaseBlocState> extends Bloc<E, S> {
  BaseBlocDelegate(S initialState) : super(initialState);
	
   // 导航器对象
  late final AppNavigator navigator;
}
```
然后在 `BasePageState` 中给当前页面绑定的 `bloc` 的 `navigator` 赋值。那么 `BasePageState`  修改后的代码如下：

```dart
// 自定义的 State 基类
abstract class BasePageState<T extends StatefulWidget,
    B extends BaseBloc> extends State<T> {
  // 导航器对象
  late final AppNavigator navigator = GetIt.instance.get<AppNavigator>();
  
  // 给当前页面绑定的 `bloc` 的 `navigator` 赋值
  late final B bloc = GetIt.instance.get<B>()
  	..navigator = navigator;
}
```
这样任何继承自 `BaseBloc` 的子类都可以使用 `navigator` 来实现页面跳转逻辑。在 `bloc` 处理完事件和状态变化后直接决定何时以及如何进行页面跳转，从而保持应用的逻辑一致性。


## 小结

本篇文章有的地方描述的比较抽象，加上大量的代码，还有前面该系列文章的内容做基础，阅读起来是有一定的难度，感兴趣的同学强烈建议直接去看项目的 [demo](https://github.com/joedrm/jianyue_music_player)，里面有更加详细的实现，结合本文的介绍更好理解。好了，今天就分享到这里，感谢您的阅读，记得关注加点赞。