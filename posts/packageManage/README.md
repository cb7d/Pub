---
title: "cocoapods"
date: 2018-06-16T10:46:42+08:00
showDate: true
draft: false
tags: ["blog","iOS","ObjC","Swift","Package"]
---

# CocoaPods

CocoaPods是OS X和iOS下的一个第三类库管理工具，通过CocoaPods工具我们可以为项目添加被称为“Pods”的依赖库（这些类库必须是CocoaPods本身所支持的），并且可以轻松管理其版本。

使用CocoaPods有以下几点好处：

- 在引入第三方库时它可以自动为我们完成各种各样的配置，包括配置编译阶段、连接器选项、甚至是ARC环境下的一些配置等。
- 使用CocoaPods可以很方便地管理的第三方SDK，大部分稳定好用的SDK都支持cocoapods导入。
- 在项目模块化的过程中方便我们模块间解耦。

### 安装

```bash
sudo gem install cocoapods
```

### 查看版本

```bash
pod --version
```

在开发中安装使用cocoapods要注意版本,因为一般开发过程中要大家一起使用同一个工程,一般为了指定版本我们会在工程下创建Gemfile来指定使用cocoapods的版本。

### 指定使用Cocoapods的版本

除了指定Gemfile以外 ， 我们还可以安装指定版本的pods

```bash
sudo gem install cocoapods -v 1.3.1
```

再查看一下pod版本我们就会发现已经安装了1.3.1

### 卸载掉不需要的版本

当我们本地同时存在多个版本的pod的时候可以把多余的卸载掉

```bash
sudo gem uninstall cocoapods
```

会提示我们选择卸载的版本

```bash
Select gem to uninstall:
 1. cocoapods-1.2.1
 2. cocoapods-1.3.1
 3. All versions
>
```

我们选择想要卸载的版本的序号就好了 。

### 使用rvm管理ruby环境

有时我们需要不同的ruby环境，而且不想更改系统自带的时候可以考虑使用rvm管理本地的ruby版本

```sh
\curl -sSL https://get.rvm.io | bash -s stable 
```

查看可用的ruby版本

```sh
rvm list known
```

使用某个制定版本作为默认的ruby版本

```
rvm use 2.6.3 --default
```

### 安装和使用bundle

```sh
gem install bundler
```

然后在工程目录下创建Gemfile

```ruby
source 'https://rubygems.org'
gem "cocoapods", "1.7.5"
```

再次使用的时候就可以使用以下方式进行包更新操作了，这样在我们同一ruby环境下也可以使用不同版本的pod

```sh
bundle exec pod install 
```

### 查看某个SDK的详细信息

cocoapods支持我们去查找想要使用的仓库 , 比如我们想查找ReactoveObjC这个库

```bash
pod spec cat ReactiveObjC
```

我们可以看到该仓库的配置信息。

```bash
{
  "name": "ReactiveObjC",
  "version": "3.1.0",
  "summary": "The 2.x ReactiveCocoa Objective-C API: Streams of values over time",
  "description": "ReactiveObjC (formally ReactiveCocoa or RAC) is an Objective-C\nframework inspired by [Functional Reactive Programming](\nhttp://en.wikipedia.org/wiki/Functional_reactive_programming).\nIt provides APIs for composing and **transforming streams of values**.",
  "homepage": "https://reactivecocoa.io",
  "screenshots": "https://reactivecocoa.io/img/logo.png",
  "license": {
    "type": "MIT",
    "file": "LICENSE.md"
  },
  "documentation_url": "https://github.com/ReactiveCocoa/ReactiveObjC/tree/master/Documentation#readme",
  "authors": "ReactiveCocoa",
  "social_media_url": "https://twitter.com/ReactiveCocoa",
  "platforms": {
    "ios": "8.0",
    "osx": "10.9",
    "watchos": "2.0",
    "tvos": "9.0"
  },
  "source": {
    "git": "https://github.com/ReactiveCocoa/ReactiveObjC.git",
    "tag": "3.1.0"
  },
  "source_files": [
    "ReactiveObjC/*.{h,m,d}",
    "ReactiveObjC/extobjc/*.{h,m}"
  ],
  "private_header_files": [
    "**/*Private.h",
    "**/*EXTRuntimeExtensions.h",
    "**/RACEmpty*.h"
  ],
  "ios": {
    "exclude_files": "ReactiveObjC/**/*{AppKit,NSControl,NSText,NSTable}*"
  },
  "osx": {
    "exclude_files": "ReactiveObjC/**/*{UIActionSheet,UIAlertView,UIBarButtonItem,UIButton,UICollectionReusableView,UIControl,UIDatePicker,UIGestureRecognizer,UIImagePicker,UIRefreshControl,UISegmentedControl,UISlider,UIStepper,UISwitch,UITableViewCell,UITableViewHeaderFooterView,UIText,MK}*"
  },
  "tvos": {
    "exclude_files": "ReactiveObjC/**/*{AppKit,NSControl,NSText,NSTable,UIActionSheet,UIAlertView,UIDatePicker,UIImagePicker,UIRefreshControl,UISlider,UIStepper,UISwitch,MK}*"
  },
  "watchos": {
    "exclude_files": "ReactiveObjC/**/*{UIActionSheet,UIAlertView,UIBarButtonItem,UIButton,UICollectionReusableView,UIControl,UIDatePicker,UIGestureRecognizer,UIImagePicker,UIRefreshControl,UISegmentedControl,UISlider,UIStepper,UISwitch,UITableViewCell,UITableViewHeaderFooterView,UIText,MK,AppKit,NSControl,NSText,NSTable,NSURLConnection}*"
```

### 开始使用

首先 ， 新建一个singlgViewApp ，然后在命令行进入该project目录

```bash
pod init
```

我们可以看到cocoapods为我们生成了一个Podfile
platform表示这个工程安装的设备，后面是系统最低版本
现在我们可以在里面添加一下刚才搜索的仓库

```ruby
# Uncomment the next line to define a global platform for your project
platform :ios, '9.0'

target 'ocTest' do
  # Uncomment the next line if you're using Swift or would like to use dynamic frameworks
  # use_frameworks!
  pod 'ReactiveObjC'
  # Pods for ocTest
end
```

这里我们添加了一个仓库,接下来再命令行执行pod update来安装所需要的仓库,安装完毕后我们可以看到当前路径下有两个工程文件 

- Test.xcodeproj
- Test.xcworkspace

我们在使用了cocoapods管理第三方库的时候,每次install或者update的时候就会生成*.xcworkspace这个文件我们需要使用这个工程进行开发调试。
现在打开工程，定位到viewController.m中就可以import并使用ReactiveObjC啦 。

### 如何使用私有源

公司内部有自己搭建的gitlab服务时，有的公司搭建的gitlab服务为了安全并没有提供外网接口，我们只能在内网访问，这时候就要在podfile中添加私有源的地址

```bash
source 'http://xx.xxxx.com/ios/cocoapods-spec.git'
source 'https://github.com/CocoaPods/Specs.git'  # 官方库
```

上面那个就是我们要添加的私有源地址，下面的是官方源地址，如果都不写的话那么默认就会使用官方源 。

### 如何创建并发布仓库到私有repo

- 首先执行命令，后面替换为你要发布SDK的名字

```bash
pod lib create TDFCommonUtil
```

<br>

- pod会给出一些选项让我们选择

```bash
To get you started we need to ask a few questions, this should only take a minute.

If this is your first time we recommend running through with the guide:
 - https://guides.cocoapods.org/making/using-pod-lib-create.html
 ( hold cmd and click links to open in a browser. )


What platform do you want to use?? [ iOS / macOS ]
 >
ios
What language do you want to use?? [ Swift / ObjC ]
 >
swift
Would you like to include a demo application with your library? [ Yes / No ]
 >
yes
Which testing frameworks will you use? [ Quick / None ]
 > None

Would you like to do view based testing? [ Yes / No ]
 > NO
```

现在我们需要添加一些信息

cd进入刚才创建的目录，执行命令 `ls` 可以看到控制台有以下输出。

```bash
Example               README.md           TDFCommonUtil.podspec
LICENSE               TDFCommonUtil         _Pods.xcodeproj
```

解释一下这里都是啥

- Example 这里面就是示例工程
- README.md 这里面是我们对改pod功能和使用等的介绍 ， 使用markdown语法
- TDFCommonUtil.podspec 这里是改模块的全部基本信息
- LICENSE 这里放的是改开源项目遵守的协议
- TDFCommonUtil 这里放着全部的类文件和资源文件
- _Pods.xcodeproj 这是示例工程的快捷方式

我们首先要为我们的pod创建一个实际的git仓库 。

打开gitlab服务创建一个空的git仓库 然后先把pod目录提交到我们的仓库 blalblabla

打开TDFCommonUtil.podspec文件

```ruby
Pod::Spec.new do |s|
  s.name             = 'TDFCommonUtil'
  s.version          = '0.1.0'
  s.summary          = 'A short description of TDFCommonUtil.'



  s.description      = <<-DESC
TODO: Add long description of the pod here.
                       DESC

  s.homepage         = 'https://github.com/xxxxx@yeah.net/TDFCommonUtil'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'xxxxx' => 'xxxxx@2dfire.com' }
  s.source           = { :git => 'https://github.com/xxxxx@yeah.net/TDFCommonUtil.git', :tag => s.version.to_s }

  s.ios.deployment_target = '8.0'

  s.source_files = 'TDFCommonUtil/Classes/**/*'
  
  # s.resource_bundles = {
  #   'TDFCommonUtil' => ['TDFCommonUtil/Assets/*.png']
  # }

  # s.public_header_files = 'Pod/Classes/**/*.h'
  # s.frameworks = 'UIKit', 'MapKit'
  # s.dependency 'AFNetworking', '~> 2.3'
end
```

这里的配置项就是生成pod需要配置的大部分选项了 ，简单介绍一下

- name pod名称
- version 版本号 ，特别注意下在更新时需要对git仓库打tag
- summary 简介
- description 描述
- homepage 主页
- license 使用协议
- author 作者
- deployment_target 需要系统版本
- source_files 源文件路径
- resource_bundles 资源文件路径 注意这里会把资源文件打包成bundle ， 调用的时候要注意下
- public_header_files 公开的头文件 （有些头文件我们不想要外部看见可以在这里去掉）
- frameworks 需要依赖的framework库
- dependency 需要依赖的其他pod

现在把这些填好吧 。

### 如何添加依赖

有的时候我们开发的pod仓库需要依赖其他仓库，比如我们需要依赖ReactiveObjC
这里就可以在TDFCommonUtil.podspec下面添加这一行

```bash
s.dependency 'ReactiveObjC'
```

ps ，这里可以指向特定的版本也可以用 '~> 2.3' 的形式表示依赖此仓库至少大于2.3版本但是不会超过3.0 。

### 使用lint命令检测我们的仓库是否还有问题

一切就绪后你可以在里面创建你的文件，添加代码了，别忘了再有文件的增加或删除后在运行一遍 ‘pod install’。

先随便创建几个文件，然后我们使用cocoapods检测我们的库

```bash
pod lib lint --sources='git@git.xxx.com:ios/cocoapods-spec.git' --use-libraries --allow-warnings --verbose --no-clean
```

这里的sources填写你所使用的私有gitlab服务，然后我们就可以静静的看着命令行了。

最后如果出现

```bash
Test passed validation.
```

证明你的库是可运行的，如果没有出现passed就注意下输出中的error信息，搜索一下error看是什么导致的 。

### 向私有源推送

lint通过后我们就可以把自己的仓库信息推送到私有源了，注意不是「仓库」是「仓库信息」，也就是x.podspec 。
cocoapod可以自动帮我们完成这件事情

```bash
pod repo push xxx-cocoapods-spec TDFOpenShopSDK.podspec --sources=git@git.xxx.com:ios/cocoapods-spec.git --allow-warnings --use-libraries --verbose
```

## 其他

### 缓存

删除指定Pod的缓存

```
 pod cache clean [NAME]
```

删除全部缓存

```
 pod cache clean --all
```

### 卸载Pod

```sh
pod deintegrate
```

### 查看环境

```
pod env 
```

可以看到Git，ruby，Xcode版本等信息

~~~sh

### Stack

```
   CocoaPods : 1.7.5
        Ruby : ruby 2.6.3p62 (2019-04-16 revision 67580) [x86_64-darwin18]
    RubyGems : 3.0.4
        Host : Mac OS X 10.14.6 (18G87)
       Xcode : 10.3 (10G8)
         Git : git version 2.20.1 (Apple Git-117)
Ruby lib dir : /Users/felix/.rvm/rubies/ruby-2.6.3/lib
Repositories : 2dfire-cocoapods-spec - git@git.2dfire.net:ios/cocoapods-spec.git @ cfd5c16d38593af16fdaa4bf1bebcf47f14b801d
               2dfire-cocoapods-spec-binary - git@git.2dfire.net:ios/cocoapods-spec-binary.git @ d07163e6b83c0fbfc71fe43a68dd2fce5f51d2b9
               2dfire-ios-cocoapods-spec - http://git.2dfire.net/ios/cocoapods-spec @ cfd5c16d38593af16fdaa4bf1bebcf47f14b801d
               master - https://github.com/CocoaPods/Specs.git @ f21043d7a7fd59154e6cae2ef819d725de394cfa
```

### Installation Source

```
Executable Path: /Users/felix/.rvm/gems/ruby-2.6.3/bin/pod
```

### Plugins

```
cocoapods-deintegrate : 1.0.4
cocoapods-plugins     : 1.0.0
cocoapods-search      : 1.0.0
cocoapods-stats       : 1.1.0
cocoapods-trunk       : 1.3.1
cocoapods-try         : 1.1.0
```
~~~

### 尝试Pod

也就是预览模式，在有多个实例工程可用的时候会让用户选择预览哪一个

```
pod try YYModel
```

### 将你的工程打包为framework

需要用到 **cocoapods-packager**

```
gem install cocoapods-packager
```

```
pod package XXX.podspec --force --dynamic --no-mangle --spec-sources=https://github.com/CocoaPods/Specs.git
```

具体参数解释

```
Usage:

    $ pod package NAME [SOURCE]

      Package a podspec into a static library.

Options:

    --force                                                         Overwrite existing
                                                                    files.
    --no-mangle                                                     Do not mangle
                                                                    symbols of
                                                                    depedendant Pods.
    --embedded                                                      Generate embedded
                                                                    frameworks.
    --library                                                       Generate static
                                                                    libraries.
    --dynamic                                                       Generate dynamic
                                                                    framework.
    --bundle-identifier                                             Bundle identifier
                                                                    for dynamic
                                                                    framework
    --exclude-deps                                                  Exclude symbols
                                                                    from dependencies.
    --configuration                                                 Build the
                                                                    specified
                                                                    configuration
                                                                    (e.g. Debug).
                                                                    Defaults to
                                                                    Release
    --subspecs                                                      Only include the
                                                                    given subspecs
    --spec-sources=private,https://github.com/CocoaPods/Specs.git   The sources to
                                                                    pull dependant
                                                                    pods from
                                                                    (defaults to
                                                                    https://github.com/CocoaPods/Specs.git)
```

