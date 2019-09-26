## Optimizing App Startup Time

启动时间耗时影响用户体验

怎样测量

main前

fork和execve

fork功能创建一个进程

execve功能加载和运行程序

添加 DYLD_PRINT_STATISTICS 运行参数

```
Total pre-main time: 430.43 milliseconds (100.0%)
         dylib loading time:  69.35 milliseconds (16.1%)
        rebase/binding time:  38.29 milliseconds (8.8%)
            ObjC setup time:  34.95 milliseconds (8.1%)
           initializer time: 287.81 milliseconds (66.8%)
           slowest intializers :
             libSystem.B.dylib :   5.42 milliseconds (1.2%)
    libMainThreadChecker.dylib :  31.36 milliseconds (7.2%)
        TDFTestProject_Example : 371.44 milliseconds (86.2%)
```

dyld 动态链接器

作用： 加载动态链接库

动态链接库的加载过程主要由dyld来完成，dyld是苹果的动态链接器
系统先读取App的可执行文件（Mach-O文件），从里面获得dyld的路径，然后加载dyld，dyld去初始化运行环境，开启缓存策略，加载程序相关依赖库(其中也包含我们的可执行文件)，并对这些库进行链接，最后调用每个依赖库的初始化方法，在这一步，runtime被初始化。当所有依赖库的初始化后，轮到最后一位(程序可执行文件)进行初始化，在这时runtime会对项目中所有类进行类结构初始化，然后调用所有的load方法。最后dyld返回main函数地址，main函数被调用，我们便来到了熟悉的程序入口。

main后

统计从main函数调用到首页viewdidload的时间

如何优化

main前

减少非系统库的依赖

合并动态库

rebase/binding 阶段

减少类和方法的数量，抽象，删除无用代码

initializer

减少在 + load 方法中的逻辑，最好能移动到+`initialize`中

main后

延时处理 didFinishLaunchingWithOptions 中的配置项

总结