# Internationalization

> 随着项目越来越成熟，逐渐拓展到海外市场，我们就需要适配多种国际化和地区、需要对自己的产品进行国际化，让更多的用户可以使用我们的APP，这就需要对我们的产品进行国际化了。在这里就介绍一下自己在国际化项目里面踩过的一些坑。

### Project配置

- 打开你的工程 ， 在左侧栏选中project
- 在打开的面板中选中project下的蓝色图标
- 找到Localizations选项 ， 添加你需要国家化的语言
- 在主工程下command + N 新建Localizable.string文件
- 选中你创建的string文件 ， 打开右侧面板
- 点击Localization组下的Localize按钮
- 把你刚才配置在工程内的选项添加进去就好。

### 调用方式

在以前调用的地方我们需要替换为本地化调用方式
比如以前我们是这么调用的：

```objectivec
NSString *tips = @"wait";
```

现在要换成

```objectivec
NSString *tips = NSLocalizedString(@"wait", nil);
```

(ps 第二个是为了方便翻译人员理解上下文语境使用的 。)

然后在stirng文件内添加对应的字符串：

```objectivec
"wait" = "别着急";
```

你可以在不同的文件添加对应不同语言的翻译 。

让我们看下这个宏的定义 ：

```objectivec
#define NSLocalizedString(key, comment) \
    [NSBundle.mainBundle localizedStringForKey:(key) value:@"" table:nil]
#define NSLocalizedStringFromTable(key, tbl, comment) \
    [NSBundle.mainBundle localizedStringForKey:(key) value:@"" table:(tbl)]
#define NSLocalizedStringFromTableInBundle(key, tbl, bundle, comment) \
    [bundle localizedStringForKey:(key) value:@"" table:(tbl)]
#define NSLocalizedStringWithDefaultValue(key, tbl, bundle, val, comment) \
    [bundle localizedStringForKey:(key) value:(val) table:(tbl)]
```

可以看到实际上就是从mainBundle中取出了指定的String文件 ， 然后根据我们在代码中定义的 ‘key’ 值取出value

有的时候我们要指定我们的string文件名字而不使用上面的那个默认名字 。

```objectivec
NSString *tips = NSLocalizedStringFromTableInBundle(@"wait", @"TDFOSLocalizable", [NSBundle bundleForClass:[self class]], @"some tips to tell user wait")；
```

这里面多出来两个参数,第一个是我们要指定的string文件名字,第二个就是要从哪个bundle中取,这个bundle的问题我们下面就会讲到。

### 使用脚本替换工程内部调用方式

那么在我们工程很大的情况下我们要把全部的字符串都替换为前缀为NSLocalizedString的形式人工手动替换肯定是不行的,又慢又不安全,没准复制粘贴的时候还把原来的key改错了,这里贴一段python的实例代码,可以稍加改动运行替换工程中的string前缀 。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os
import re
import sys
import datetime

folderPath = '/Users/felix/Documents/2dfire/TDFOpenShop/TDFOpenShop/Classes'
stringToReplace = 'NSLocalizedString'
stringReplaceTo = 'TDFOSLocalizedString'

def scan_files(rootdir):
    files = []
    for  parent, dirnames, filenames  in  os.walk(rootdir):
        for  filename  in  filenames:
            if len(filename.split('.',1))>1:
                if filename.split('.',1)[1] == 'm':
                    files.append(os.path.join(parent, filename))
    return files

def replaceStringIn(file):
    code = open(file,'r+')
    text = code.read()
    code.seek(0,0)
    code.write(text.replace(stringToReplace,stringReplaceTo))
    code.close()


def main():
    files = scan_files(folderPath)
    for filePath in files:
        replaceStringIn(filePath)

if __name__ == '__main__':
    main()
```

这里面我是把以前的NS开头的宏替换为自定义的宏 ， 大同小异 ， 可以参照这个修改下即可 。

### 编译运行测试

当我们全部修改好以后肯定是要测试下显示对不对的、有两种方法可以快速切换App语言

- 打开模拟器或真机 general -> setting -> lang ,不过这样比较繁琐。
- 点击导航栏上的模拟器图标左边的schema选项,选择editSchema -> Run -> Options ->Application Language 直接改为你需要验证的语言运行就可以啦 。

### 如何在模块内单独配置国际化文件

很多时候我们的工程体量大到一定程度的时候都会模块化掉,几个人或者一个人负责一个模块,而有的模块是要作为SDK提供外部interface使用的 。当你把SDK提供出去的时候,我们期望的效果是在其他人（其他工程）内部运行的、我们的SDK也能够国际化显示 。但是在我们整个工程的string文件包含几千上万条的时候显然不需要全部移动到模块内 。

所以在这里面我们要做的就是把需要的使用到的字符串和翻译好的value从主工程迁移到模块内部 。

- 解决方案：使用cocoapods集成的模块化内部进行国际化

我们先把主工程的国际化文件的文件夹复制到我们的模块路径下面,在模块内部有一个配置文件 `x.podspec` 里面添加这样一段配置

```ruby
s.resources  = 'xxx/*.{xib,jpg,png,xcassets}','xxx/*.lproj'
```

主要注意后面 , 这里就把我们刚复制来的国际化问价加入了模块的资源中 ，这时 ，如果我们不想用跟主工程一样的名字 ，Localizable.strings ，可以把这个名字也给改掉 ， 比如 abc.strings

- 调用

这里为了有别于主工程的国际化 ， 我们把strings文件改名为 abc.strings 然后在模块内新建一个头文件

```objectivec
#ifndef TDFOSLocalizeMacro_h
#define TDFOSLocalizeMacro_h

#define TDFOSLocalizedString(key, comment) \
NSLocalizedStringFromTableInBundle(key, @"abc", [NSBundle bundleForClass:[self class]], comment)

#endif /* TDFOSLocalizeMacro_h */
```

这里可以使用上面的脚本再次替换字符串前缀,然后运行下看看是否替换成功。
这里注意要重新pod install下,否则cocoapods不会帮我们把资源引入包内 。
ps: 别忘了删掉主工程中移除来的字符串 ！！！

### 总结

基本用法暂时介绍到这里 ， 关于模块化和国际化的东西要多了解一下NSbundle