上篇文章讲的是状态管理，提到了 `Flutter BLoC` ，相比与原生的 `setState()` 及`Provider`等有哪些优缺点，并结合实际项目写了一个简单的使用，接下来本篇文章来讲 `Flutter` 大型项目是如何进行分层设计的，费话不多说，直接进入正题哈。

## 为啥需要分层设计
其实这个没有啥固定答案，也许只是因为某一天看到手里的代码如同屎山一样，如下图，而随着业务功能的增加，不停的往这上面堆，这个屎山也会愈发庞大和混乱，如果这样继续下去，直到某一天因为一个小小的Bug，你需要花半天的时间来排查问题出在哪里，最后当你觉得问题终于改好了的时候，却不料碰了不该碰的地方，结果就是 `fixing 1 bug will create 10 new bugs`，甚至程序的崩溃。

![](https://s2.loli.net/2024/03/26/JeSiB7lAUKF69zg.png)

随着这种问题的凸显，于是团队里的显眼包A提出了要求团队里的每个人都必须负责完成给自己写的代码添加注释和文档，规范命名等措施，一段时间后，发现代码是规范了，但问题依然存在，这时候才发现如果工程的架构分层没有做好，再规范的代码和注释也只是在屎山上雕花，治标不治本而已。

![](https://s2.loli.net/2024/03/26/r4lYmkcz7WX9sDG.png)

请原谅我打了一个这么俗的比方，但话糙理不糙，那么啥是应用的分层设计呢？

简单的来说，应用的分层设计是一种将应用程序划分为不同层级的方法，每个层级负责特定的功能或责任。其中表示层（`Presentation Layer`）负责用户界面和用户交互，将数据呈现给用户并接收用户输入；业务逻辑层（`Business Logic Layer`）处理应用程序的业务逻辑，包括数据验证、处理和转换；数据访问层（`Data Access Layer`）负责与数据存储交互，包括数据库或文件系统的读取和写入操作。

![](https://s2.loli.net/2024/03/26/wKE4U3AncJN1XHy.png)

这样做有什么好处呢？一句话总结就是为了让代码层级责任清晰，维护、扩展和重用方便，每个模块能独立开发、测试和修改。

## `App` 原生开发的分层设计

说到 `iOS`、`Android` 的分层设计，就会想到如 `MVC`、`MVVM` 等，它们主要是围绕着控制器层（`Controller`）、视图层（`View`）、和数据层（`Model`），还有连接 `View` 和 `Model` 之间的模型视图层（`ViewModel`）这些来讲的。

![MVVM](https://s2.loli.net/2024/03/26/hMzWOpvum4jetRy.png)

然而，`MVC`、`MVVM` 概念还不算完整的分层架构，它们只是关注的 `App` 分层设计当中的应用层（`Applicaiton Layer`）组织方式，对于一个简单规模较小的App来说，可能单单一个应用层就能搞定，不用担心业务增量和复杂度上升对后期开发的压力，而一旦 `App` 上了规模之后就有点应付不过来了。

当 `App` 有了一定规模之后，必然会涉及到分层的设计，还有模块化、`Hybrid` 机制、数据库、跨项目开发等等，拿 `iOS` 的原生分层设计落地实践来说，通常会将工程拆分成多个`Pod`私有库组件，拆分的标准视情况而定，每一个分层组件是独立的开发和测试，再在主工程添加` Pod` 私有库依赖来做分层设计开发。

此处应该有 `Pod` 分层组件化设计的配图，但是太懒了，就没有一个个的去搭建新项目和 `Pod` 私有库，不过 `iOS` 原生分层设计不是本篇文章的重点，本篇主要谈论的是 `Flutter App` 的分层设计。


## `Flutter` 的分层设计

分层架构设计的理念其实是相通的，差别在于语言的特性和具体项目实施上，`Flutter` 项目也是如此。试想一下，当各种逻辑混合在一次的时候，即便是选择了像 `Bloc` 这样的状态管理框架来隔离视图层和逻辑实现层，也很难轻松的增强代码的拓展性，这时候选择采用一个干净的分层架构就显得尤为重要，怎样做到这一点呢，就需要**将代码分成独立的层，并依赖于抽象而不是具体的实现。**

![分层架构设计](https://s2.loli.net/2024/03/26/H1YMksuT5fyQBXo.png)

`Flutter App` 想要实现分层设计，就不得不提到包管理工具，如果在将所有分层组件代码放在主工程里面，那样并不能达到每个组件单独开发、维护和测试的目的，而如果放在新建的 `Dart Package` 中，没发跨多个组件改代码和测试，无法实现本地包链接和安装。使用 [melos](https://melos.invertase.dev/) 就能解决这个问题，类似于 `iOS` 包管理工具 `Pod`, 而 `melos`  是 `Flutter` 项目的包管理工具。


## 组件包管理工具

1. 安装 `Melos`，将 `Melos` 安装为全局包，这样整个系统环境都可以使用：
  
    ```
    dart pub global activate melos
    ```
    
2. 创建 `workspace` 文件夹，我这里命名为 `flutter_architecture_design`，添加 `melos`  的配置文件`melos.yaml` 和 `pubspec.yaml`，其目录结构大概是这样的：

    ```text
    flutter_architecture_design
    ├── melos.yaml
    ├── pubspec.yaml
    └── README.md
    ```
    
 3. 新建组件，以开发工具 `Android Studio` 为例，选择 `File` -> `New` -> `New Flutter Project`，根据需要创建组件包，需要注意的是组件包存放的位置要放在 `workspace` 目录中。
 ![新建组件](https://s2.loli.net/2024/03/26/pBbWe9kSX67xHnR.png)
 
 4. 编辑 `melos.yaml` 配置文件，将上一步新建的组件包名放在 `packages` 之下，添加 `scripts` 相关命令，其目的请看下一步：
    
    ```yaml
    name: flutter_architecture_design

    packages:
      - widgets/**
      - shared/**
      - data/**
      - initializer/**
      - domain/**
      - resources/**
      - app/**

    command:
      bootstrap:
        usePubspecOverrides: true

    scripts:
      analyze:
        run: dart pub global run melos exec --flutter "flutter analyze --no-pub --suppress-analytics"
        description: Run analyze.

      pub_get:
        run: dart pub global run melos exec --flutter "flutter pub get"
        description: pub get

      build_all:
        run: dart pub global run melos exec --depends-on="build_runner" "flutter packages pub run build_runner build --delete-conflicting-outputs"
        description: build_runner build all modules.

      build_data:
        run: dart pub global run melos exec --fail-fast --scope="*data*" --depends-on="build_runner" "flutter packages pub run build_runner build --delete-conflicting-outputs"
        description: build_runner build data module.

      build_domain:
        run: dart pub global run melos exec --fail-fast --scope="*domain*" --depends-on="build_runner" "flutter packages pub run build_runner build --delete-conflicting-outputs"
        description: build_runner build domain module.

      build_app:
        run: dart pub global run melos exec --fail-fast --scope="*app*" --depends-on="build_runner" "flutter packages pub run build_runner build --delete-conflicting-outputs"
        description: build_runner build app module.

      build_shared:
        run: dart pub global run melos exec --fail-fast --scope="*shared*" --depends-on="build_runner" "flutter packages pub run build_runner build --delete-conflicting-outputs"
        description: build_runner build shared module.

      build_widgets:
        run: dart pub global run melos exec --fail-fast --scope="*widgets*" --depends-on="build_runner" "flutter packages pub run build_runner build --delete-conflicting-outputs"
        description: build_runner build shared module.
    ```
   
   5. 打开命令行，切换到 `workspace` 目录，也就是 `flutter_architecture_design` 目录，执行命令。
       
       ```
       melos bootstrap
       ```
       
       出现 `SUCCESS` 之后，现在的目录结构是这样的：
       ![目录结构](https://s2.loli.net/2024/03/26/UnmYS4u1hlGaOX2.png)
      
  6. 点击` Android Studio` 的 `add configuration`，将下图中的 `Shell Scripts` 选中后点击 `OK`。
  ![Add Shell Scripts](https://s2.loli.net/2024/03/26/CtvHznscBp31xOl.png)
  以上的 `Scripts` 添加完后就可以在这里看到了，操作起来也很方便，不需要去命令行那里执行命令。
  ![Shell Scripts](https://s2.loli.net/2024/03/26/Ls2BmZ98flTNu6X.png)
  
## `Flutter` 分层设计实践
  
接下来介绍一下上面创建的几个组件库。
- `app`：项目的主工程，存放业务逻辑代码、 `UI` 页面和 `Bloc`，还有 `styles`、`colors` 等等。
- `domain`：实体类（`entity`）组件包，还有一些接口类，如 `repository`、`usercase`等。
- `data`：数据提供组件包，主要有：`api_request`，`database`、`shared_preference`等，该组件包所有的调用实现都在 `domain` 中接口 `repository` 的实现类 `repository_impl` 中。
- `shared`：工具类组件包，包括：`util`、`helper`、`enum`、`constants`、`exception`、`mixins`等等。
- `resources`：资源类组件包，有`intl`、公共的`images`等
- `initializer`：模块初始化组件包。
- `widgets`：公共的 `UI` 组件包，如常用的：`alert`、`button`、`toast`、`slider` 等等。

它们之间的调用关系如下图：

![Flutter App Architecture Design](https://s2.loli.net/2024/03/27/RvQMhaPd8p4cOf5.png)
       
其中 `shared` 和 `resources` 作为基础组件包，本身不依赖任何组件，而是给其它组件包提供支持。

作为主工程 `App` 也不会直接依赖 `data` 组件包，其调用是通过 `domain` 组件包中 `UseCase` 来实现，在 `UseCase`  会获取数据、处理列表数据的分页、参数校验、异常处理等等，获取数据是通过调用抽象类 `repository` 中相关函数，而不是直接调用具体实现类，此时`App` 的 `pubspec.yaml` 中配置是这样的：

```yaml
name: app
description: A new Flutter project.
publish_to: 'none' # Remove this line if you wish to publish to pub.dev
version: 1.0.0+1

environment:
  sdk: ">=2.17.0 <3.0.0"

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.2
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

dev_dependencies:
  flutter_test:
    sdk: flutter

flutter:
  uses-material-design: true
  generate: false
  assets:
    - assets/images/
```

提供的数据组件包 `data` 实现了抽象类 `repository` 中相关函数，只负责调用 Api 接口获取数据，或者从数据库获取数据。当上层调用的时候不需要关心数据是从哪里来的，全部交给 `data` 组件包负责。

`initializer` 作为模块初始化组件包，仅有一个 `AppInitializer` 类，其主要目的是将其它的模块的初始化收集起来放在 `AppInitializer` 类中 `init()` 函数中，然后在主工程入口函数：`main()` 调用这个 `init()` 函数，常见的初始化如：`GetIt` 初始化、数据库 `objectbox` 初始化、`SharedPreferences`初始化，这些相关的初始会分布在各自的组件包中。

```dart
class AppInitializer {
  AppInitializer();

  Future<void> init() async {
    await SharedConfig.getInstance().init();
    await DataConfig.getInstance().init();
    await DomainConfig.getInstance().init();
  }
}
```
`widgets` 作为公共的 UI 组件库，不处理业务逻辑，在多项目开发时经常会使用到。上图中的 `Other Plugin Module` 指的的是其它组件包，特别是需要单独开发与原生交互的插件时会用到，

这种分层设计出来的架构或许在开发过程中带来一下不便，如调用一个接口，第一步：需要先在抽象类 `repository` 写好函数声明；第二步：然后再去`Api Service` 写具体请求代码，并在`repository_impl` 实现类中调用；第三步：还需要在 `UserCase` 去做业务调用，错误处理等；最后一步：在`bloc`的`event`中调用。这么一趟下来，确实有些繁琐或者说是过度设计。但是如果维度设定在大的项目中多人合作开发的时候，却能规避很多问题，每个分层组件都有自己的职责互不干扰，都支持单独的开发测试，尽可能的做到依赖于抽象而不是具体的实现。