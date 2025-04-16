前两篇文章说到了 **[状态管理](https://www.nnxkcloud.com/2023/08/09/flutter-%E5%A4%A7%E5%9E%8B%E9%A1%B9%E7%9B%AE%E6%9E%B6%E6%9E%84%EF%BC%9A%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86%E7%AF%87/)** 和 **[分层设计](https://www.nnxkcloud.com/2024/03/26/flutter%E5%A4%A7%E5%9E%8B%E9%A1%B9%E7%9B%AE%E6%9E%B6%E6%9E%84%EF%BC%9A%E5%88%86%E5%B1%82%E8%AE%BE%E8%AE%A1%E7%AF%87/)** ，本篇换个角度来讲讲 `Flutter` 中的依赖管理，需要注意的是这里讲的依赖管理主要指项目内的代码，包括依赖注入、组件包之间的依赖关系，而不是第三方库的依赖管理，下面开始进入正题。

## 为啥需要依赖管理
你可能会问，为啥搞那么麻烦使用依赖管理？不就是实例化的时候多写几句代码嘛，没有用它照样能把功能完成，不是吗？这么说也没错，如果只是在项目的几个位置这样写，问题也不大，可是放大到大型的项目中呢，该如何应对呢？下面这段代码是没有使用依赖管理之前的：

```Dart
class RepositoryTool {
  ApiService _apiService;
  AppPreferences _appPreferences;
  AppDatabase _appDatabase;
  
  factory RepositoryTool() => _singleton;
  static final RepositoryTool _singleton = RepositoryTool._();
  static RepositoryTool get instance => RepositoryTool();
  
  RepositoryTool._() {
    _apiService = ApiService();
    _appPreferences = AppPreferences();
    _appDatabase = AppDatabase();
  }
  
  // 登录请求
  Future<void> login({
    required String username,
    required String password,
  }) async {
    _appApiService.login(username: username, password: password);
  }
  
  // 从数据库中获取用户信息
  Future<User> getLocalUser(int id) async {
    _appDatabase.getLocalUser(id);
  }
}
```
这里单例类 `RepositoryTool` 为整个应用提供数据的，其有三个成员变量：`_apiService` 负责接口请求服务；`_appPreferences` 负责从偏好设置中读取或者存储数据，主要是一些字段或者 `Bool` 值；`_appDatabase` 负责从数据库中读取或者存储数据，这三个也是应用中经常使用的，它们都和数据相关却各司其职。但是当我们需更改一下 `_apiService` 的构造函数的时候，如下面的代码：

```Dart
class ApiService {
  ApiService(
    this._noneAuthApiClient, 
    this._authApiClient);

  final NoneAuthApiClient _noneAuthApiClient;
  final AuthApiClient _authApiClient;
  
  // 登录操作
  Future<void> login({String username, String password}) async {
    await _authApiClient.request(
        method: RestMethod.post,
        path: '/v1/auth/login',
        queryParameters: {"username": username, "password": password});
	}
}

// 继承 RestApiClient，用于发送不带 Token 的接口请求，如登录操作
class NoneAuthApiClient extends RestApiClient {
  NoneAuthApiClient(HeaderInterceptor _headerInterceptor)
      : super(baseUrl: "" interceptors: [
          _headerInterceptor,
        ]);
}

// 继承 RestApiClient，用于发送带 Token 的接口请求
class AuthApiClient extends RestApiClient {
  AuthApiClient(
    HeaderInterceptor _headerInterceptor,
    AccessTokenInterceptor _accessTokenInterceptor,
    RefreshTokenInterceptor _refreshTokenInterceptor,
  ) : super(baseUrl: "", interceptors: [
          _headerInterceptor,
          _accessTokenInterceptor,
          _refreshTokenInterceptor,
        ]);
}

// Api 请求的基类
class RestApiClient {
  RestApiClient({
    this.baseUrl = '',
    this.interceptors = const [],
  })

  // 用 Dio 发送请求
  Future<T> request<T, D>(){}
}
```
这个时候又得回到 `RepositoryTool` 的 `RepositoryTool._() ` 修改实例 `_apiService` 初始化代码，而在 `ApiService` 类中的成员变量 `_noneAuthApiClient` 和 `_authApiClient` 也需要在实例化的时候传入不同的 `Interceptor` 。

同样 `_appPreferences`  和 `_appDatabase`  的构造函数也是需要传入不同类实例参数，如果不把这些参数放在构造函数中传入的话，就需要在用到地方去实例化。如 `RepositoryTool` 的 `RepositoryTool._() ` 的代码，导致这些类 `ApiService`、`AppPreferences`、`AppDatabase` 和 `RepositoryTool` 紧密耦合在一起，难以进行单元测试和扩展，而且，构造函数一改其它所有用到的地方都得改，代码维护成本太高了。

## 依赖管理是什么
用过 `get_it` 的同学可能会说，用依赖注入不就可以优化上面的问题吗，和依赖管理有什么关系呢？依赖注入（`Dependency Injection`，简称：`DI`）和依赖管理（`Dependency Management`）在软件开发中确实是两个相关但不同的概念。

依赖注入是一种软件设计模式，它通过将依赖关系从代码中分离出来，并由外部系统在运行时动态地注入到代码中，从而实现了依赖的管理，是一种实现依赖管理的技术手段。

依赖注入需要依赖管理来实现，在依赖注入的实践过程中，需要一个依赖管理的机制来管理各个组件之间的依赖关系。这包括管理依赖的声明周期、版本、配置等信息，并确保依赖的正确注入和使用。

如下图体现出来的各个组件的依赖关系：

![各个组件的依赖关系](https://s2.loli.net/2024/03/27/RvQMhaPd8p4cOf5.png)

在主工程 `App` 中的 `LoginPage` 发送登录请求，上图中是在 `Bloc` 调用抽象类 `Repository` 登录函数，`Repository` 是在 `domain` 组件包的，实际上登录操作的实现是在 `data` 组件包中的类 `RepositoryImpl` 的 `login` 函数。`App` 只需要依赖 `domain` 组件包，不直接依赖 `data`，而这之间的实现代码则是由`DI` 帮我们完成的。

## Flutter 中依赖管理方案
具体该怎么操作呢？我在 **[《Flutter开发7个建议，让你的工作效率飙升》](https://www.nnxkcloud.com/2024/04/13/flutter%E5%BC%80%E5%8F%917%E4%B8%AA%E5%BB%BA%E8%AE%AE%EF%BC%8C%E8%AE%A9%E4%BD%A0%E7%9A%84%E5%B7%A5%E4%BD%9C%E6%95%88%E7%8E%87%E9%A3%99%E5%8D%87/)** 也提到了依赖注入，也是以 `Repository` 为例，通过 `get_it` 和 `injectable` 配合来使用，达到解耦`App`  和 `data` 组件之间的依赖关系。今天换一个，我们以 `AppNavigator` 为例，来看看怎样解耦业务调用 `AppNavigator`  和导航具体实现的之间关系。先来看看 `AppNavigator` 中的代码。

```Dart
abstract class AppNavigator {
  const AppNavigator();

  int get currentBottomTab;

  Future<T?> push<T extends Object?>(AppRouteInfo appRouteInfo);
  
  Future<bool> pop<T extends Object?>({
    T? result,
    bool useRootNavigator = false,
  });
}
```
这里的 `AppNavigator` 还是 `abstract` 类，和 `Repository` 一样，也是在 `domain` 组件包中，从上面的各个组件的依赖关系图看出，`AppNavigator`  实现类 `AppNavigatorImpl` 放在了主工程 `App` 组件包中的。

类`AppNavigatorImpl` 实现代码如下：

```Dart
import 'dart:async';
import 'package:auto_route/auto_route.dart';
import 'package:domain/domain.dart';
import 'package:flutter/material.dart' as m;
import 'package:injectable/injectable.dart';

@LazySingleton(as: AppNavigator)
class AppNavigatorImpl extends AppNavigator{
  AppNavigatorImpl(
    this._appRouter,
  );

  final AppRouter _appRouter;
  
  @override
  int get currentBottomTab {
    return 0;
  }

  @override
  Future<T?> push<T extends Object?>(AppRouteInfo appRouteInfo) {
    return _appRouter.push<T>(_appRouteInfoMapper.map(appRouteInfo));
  }

  @override
  Future<bool> pop<T extends Object?>({T? result, bool useRootNavigator = false}) {
    return _appRouter.pop<T>(result);
  }
}
```
`AppNavigatorImpl` 中实现的路由导航是插件 `auto_route`，通过构造函数传过来，此时运行
```
flutter packages pub run build_runner build
```
在 `di.config.dart` 自动生成了如下代码：
```Dart
import 'package:get_it/get_it.dart' as _i1;
import 'package:injectable/injectable.dart' as _i2;
import 'package:domain/domain.dart' as _i4;
import 'package:app/navigation/routes/app_router.dart' as _i5;
import 'package:app/app.dart' as _i6;
import 'package:app/navigation/app_navigator_impl.dart' as _i7;

extension GetItInjectableX on _i1.GetIt {
  // initializes the registration of main-scope dependencies inside of GetIt
  _i1.GetIt init({
    String? environment,
    _i2.EnvironmentFilter? environmentFilter,
  }) {
    final gh = _i2.GetItHelper(
      this,
      environment,
      environmentFilter,
    );
   
    gh.lazySingleton<_i5.AppRouter>(() => _i5.AppRouter());
    gh.lazySingleton<_i4.AppNavigator>(() => _i7.AppNavigatorImpl(
          gh<_i6.AppRouter>(),
        ));
    return this;
  }
}
```
从 `DI` 生成的代码也能看出它们之间的依赖关系，在用到导航跳转的时候从 `get_it` 容器中取出使用即可：
```Dart
late final AppNavigator navigator = GetIt.instance.get<AppNavigator>();
navigator.push(...);
```
这样调用的时候是通过抽象类 `AppNavigator` ，不需要关心导航的具体是怎么实现的。或许哪天不用插件 `auto_route` 来做路由导航，只需要继承 `AppNavigator` 的并实现它的函数即可，而调用的地方不需要做任何更改。

上面的例子是以 `Navigator` 相关的作为切入点，其实在大型 `Flutter` 项目中用这种 **并依赖于抽象而不是具体的实现** 的理念来做依赖管理的地方有很多，这样做的好处是使得代码更加清晰、模块化，并且方便进行单元测试和维护。同时，通过依赖注入容器的注册和获取，可以实现依赖对象的延迟加载和单例管理，提高了应用程序的性能和效率。

本篇算是上一篇文章 **[《Flutter大型项目架构：分层设计篇》](分层设计)** 的延伸，也是做分层设计的落地实施时必不可少的环节。今天分享到这里，感谢您的阅读。