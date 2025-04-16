---
title: Flutter大型项目架构：私有组件包管理
date: 2024-07-25 17:31:35
tags:
---

随着项目功能模块越来越多，怎么去管理这些私有组件包是一个不得不面对的问题，特别对于团队开发来讲，一些通用的公共组件往往会在多个项目间使用，多的有几十个，每个组件包都有有自己的版本，组件包之间还有依赖关系，那么如何在团队内部共享私有组件包，控制依赖私有组件包的版本和组件包的更新维护，而且要确保其安全环境，也就是没有公网无法访问的安全环境，这些都是需要考虑的，如何去管理这些组件对提升开发效率和代码质量就显得尤为重要，那有日常开发中哪些的方式来管理私有组件库呢？本篇文章就来介绍一下。

<!-- more -->

### 主工程中管理组件

主工程新建一个 `packages` 目录，从主工程抽取出来的组件放在改目录下，目录结构如下图：

![4881718430073_.pic.png](https://s2.loli.net/2024/06/15/X6OpUV7PEcLM8hB.png)

这种将组件分离到 `packages` 目录下更加简单直接，也可以实现模块化管理。但是由于所有代码都放在主工程的目录下，使用同一个 `git` 仓库，对于每个组件独立开发和维护不太方便，各组件的依赖关系也不是很明了，如主工程明明不需要直接依赖于 `data` 组件，但 `data` 组件的代码却在主工程的目录下面，两者之间本应该没有任何直接关系。

### `melos` 管理组件
在之前的文章《Flutter大型项目架构：分层设计篇》中介绍了如何使用 `melos` 创建一个 `workspace` 来管理各个分层的组件包，有兴趣的话可以去翻一翻这篇文章。在大型项目中，这种方式可以让每个组件独立开发和维护，还可以借助 `melos` 编写各种编译指令，每个组件单独构建运行，组件间的依赖关系清晰明了。

![4891718432664_.pic.png](https://s2.loli.net/2024/06/15/N2Eyis6JPgHcZYQ.png)

`melos` 有一套独特的工作流程和命令方便单独建运行，同时也有一定的学习成本，尤其是对于小型项目来讲使用 `melos` 就显得有点复杂了，而且 `melos` 管理组件还是有一个同样的问题，那就是各组件和主工程使用同一个 `git` 仓库。如果想要组件使用独立的 `git` 仓库，该怎么做呢？

### `Github` 或者 `Gitlab`
将组件上传到`Github` 或者 `Gitlab` 上，使用的时候直接在 `podspec.yaml` 文件中添加依赖，日常开发中也是比较常见的。

```yaml
dependencies:
  library_name:
    git:
      url: git@github.com:username/library_name.git
      ref: dev
```

除了上面指定某个分支（如：`dev`），还可以特指标签（如 `v1.0.0`），或者具体的提交哈希值。

```yaml
dependencies:
  library_name:
   git:
    url: https://github.com/username/library_name.git
    ref: e234072340    #commit reference id
```
可以看出如果使用 `Github` 或者 `Gitlab`  来管理 `Flutter` 组件包，需要手动管理版本标签或提交哈希，这是因为 `Dart` 的 `pub` 工具不支持从 `git` 仓库自动解析版本号，只能指定具体的分支、标签或提交哈希，不能像从 `pub.dev` 获取包一样指定版本号（如 `^1.0.0`）。如果使用私有仓库，还需要付费订阅私有仓库的服务。

### 第三方平台托管

用过比较多的就是 [OnePub](https://docs.onepub.dev/) ，一个专门用于管理和发布 `Dart` 和 `Flutter` 包的第三方平台，旨在提供更灵活的包管理功能，尤其是对于私有包和团队协作来说，不仅可以将 `Dart/Flutter` 包发布到私有存储库、在项目中添加依赖，还可以与同事共享私有包
有界面来搜索公共和私有包，通过电子邮件向团队发送新包版本的通知，与 `pub` 工具无缝集，将 `OnePub` 集成到 `CI/CD` 管道中，自动化包的发布和管理过程等等功能。而且还有2个成员25个 `packages` 的范围内免费使用，是不是很香，不想去折腾的话使用 `OnePub` 还是不错的。

使用第三方平台托管还有有其它的选择，如`Cloudsmith`、`JFrog`、`JetBrains Space` 等，`Cloudsmith` 除了支持 `Dart` 和 `Flutter` 包管理和分发，平台还支持超过 20 种包格式，包括 `Docker`、`npm`、`Python`、`Maven`、`RubyGems` 等，适合使用多种语言的中、大型团队。

![5021721891883_.pic.png](https://s2.loli.net/2024/07/25/LU4k3iMZn591dYR.png)

### 搭建自定义包服务器
`dart pub` 是支持自定义的第三方包存储库的，但是自行搭建的话，要处理如安全性（数据加密认证和授权）、存储方案、`API` 的设计、插件包的版本解析和版本控制、服务器和包管理系统与最新的 `Flutter` 和 `Dart` 版本兼容性等等方面的问题，这么一套操作下来太费时费力，如果没有运维的做技术支持的话，很难推行下去，那有没有其它的现成的方案呢？

这里推荐使用之前字节开源的 `unpub`，使用 `MongoDB` 作为默认数据库，模拟了 `pub.dev` 的功能，也能托管自己的包存储库。以下代码在本地开启一个 `server`，元信息数据存储在本地的 `mongodb`，包文件存在本地的 `unpub-packages` 文件夹中。

```dart
import 'package:mongo_dart/mongo_dart.dart';
import 'package:unpub/unpub.dart' as unpub;

main(List<String> args) async {
  final db = Db('mongodb://localhost:27017/dart_pub');
  await db.open(); // make sure the MongoDB connection opened

  final app = unpub.App(
    metaStore: unpub.MongoStore(db),
    packageStore: unpub.FileStore('./unpub-packages'),
  );

  final server = await app.serve('0.0.0.0', 4000);
  print('Serving at http://${server.address.host}:${server.port}');
}
```
同时，作者还提供将服务部署到 `AWS` 的插件包，只需修改 `packageStore` 参数即可。使用自定义的包存储库时，在你的 Flutter 项目中编辑 `pubspec.yaml` 文件，添加你的自定义包服务器作为一个新的包源。

```yaml
dependencies:
  my_custom_package:
    version: ^0.0.1
    hosted:
      name: your_custom_server_name
      url: https://your-custom-server-url.com/packages
```

### 小结

以上列出来的几种组件包管理方式各有优劣，直接放在主工程中是最简单直接但是独立开发和维护不太方便；`melos` 管理组件可以独立开发维护却要学习相关的指令；`git` 能做到独立仓库管理但不支持自动解析版本；搭建自定义包服务器似乎是终极解决方案但需要额外部署和维护一个包服务器；第三方平台托管貌似没有前面面临的问题但托管包的数量有限，超出限制要收费，当然预算充足的话就不是问题。怎么选择就需要根据项目的复杂度和团队的情况，目前本人项目中大都使用的是 `melos` 来管理组件，那么你使用的是哪种方式呢？或者有没有其它更好的方式来管理项目中的组件？欢迎来分享和交流一下！