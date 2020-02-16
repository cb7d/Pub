# M-V-VM

> 目前客户端最流行的架构应该就是MVVM，然而在看了一些文章之后发现大部分是理论而并没有仔细讲解具体的架构方法和实践，这篇博客说说我在实际工作中的使用。

## 引言

提到MVVM我们不得不先来认识一下MVC：
MVC模式（Model–view–controller）是软件工程中的一种软件架构模式，把软件系统分为三个基本部分：模型（Model）、视图（View）和控制器（Controller）。MVC模式最早由Trygve Reenskaug在1978年提出[1]，是施乐帕罗奥多研究中心（Xerox PARC）在20世纪80年代为程序语言Smalltalk发明的一种软件架构。MVC模式的目的是实现一种动态的程序设计，使后续对程序的修改和扩展简化，并且使程序某一部分的重复利用成为可能。除此之外，此模式通过对复杂度的简化，使程序结构更加直观。软件系统通过对自身基本部分分离的同时也赋予了各个基本部分应有的功能。专业人员可以通过自身的专长分组：

- 控制器（Controller）- 负责转发请求，对请求进行处理。
- 视图（View） - 界面设计人员进行图形界面设计。
- 模型（Model） - 程序员编写程序应有的功能（实现算法等等）、数据库专家进行数据管理和数据库设计(可以实现具体的功能)。

### MVVM

> MVVM是Model-View-ViewModel的简写，最早是由微软公司提出并运用，是MVP（Model-View-Presenter）模式与WPF结合的应用方式时发展演变过来的一种新型架构架构。MVVM有助于将图形用户界面的开发与业务逻辑或后端逻辑（数据模型）的开发分离开来，这是通过置标语言或GUI代码实现的。MVVM的视图模型是一个值转换器，这意味着视图模型负责从模型中暴露（转换）数据对象，以便轻松管理和呈现对象。在这方面，视图模型比视图做得更多，并且处理大部分视图的显示逻辑。视图模型可以实现中介者模式，组织对视图所支持的用例集的后端逻辑的访问。

- 模型 模型是指代表真实状态内容的领域模型（面向对象），或指代表内容的数据访问层（以数据为中心）。
- 视图 就像在MVC和MVP模式中一样，视图是用户在屏幕上看到的结构、布局和外观（UI）。
- 视图模型 视图模型是暴露公共属性和命令的视图的抽象。MVVM没有MVC模式的控制器，也没有MVP模式的presenter，有的是一个绑定器。在视图模型中，绑定器在视图和数据绑定器之间进行通信。
- 绑定器 声明性数据和命令绑定隐含在MVVM模式中。在Microsoft解决方案堆中，绑定器是一种名为XAML的标记语言。绑定器使开发人员免于被迫编写样板式逻辑来同步视图模型和视图。在微软的堆之外实现时，声明性数据绑定技术的出现是实现该模式的一个关键因素。

## MVVM 的优点

### 解决controller过于臃肿

在MVC中很容易就会把一些业务逻辑，网络请求，数据IO都放在controller中  

注意,这里不是说MVC的控制器一定很臃肿，而是「容易变得臃肿」 

在我们新建一个工程的时候,苹果会自动帮我们生成一个ViewController,而在动手开始写代码的时候,往往控制不住就直接将逻辑写在Controller中。

MVVM架构会要求我们把任何与非View的逻辑玻璃出来,Controller中除了绑定viewModel之外的代码只允许出现对View的操作。因为Controller对我们来说也只是一个View。

### 逻辑分离， 易于测试

就像上面说的,业务逻辑都会抽离出来放在viewModel中 ，这样可以在任何地方重用这一堆业务 

除此之外,我们的代码将会更加易于测试,避免出现在MVC中可能出现的那种超长的方法、严重依赖全局状态导致难以测试的问题。

### View重用

在MVVM中,View只需要与ViewMode交互,不会收到其他的影响,所以不但vm、m 可以重用,view一样可以重复使用,修改的时候也更加方便。

## 缺点

### BUG与传递

由于在MVVM里面View和ViewModel是松耦合的,在测试出问题的时候就要排查各个地方的问题,

有可能是vm中的也有可能是view中的。由于vm会传递数据,一个bug会很容易的传递到其他地方,引发更大的问题。

并且其中一个地方出现问题的话,这个BUG就极有可能随着传递到其他的逻辑中,从而导致更严重的问题发生。

### 需要维护额外的开销。

额外的viewModel使用也并不是无代价的,有可能由于各种原因导致管理起来稍微复杂。而额外的,如果因为强引用或其他原因导致的循环引用等内存不能正确释放的情况下,有可能会内存疯涨，所以需要确保你的使用方式是无副作用的。

## 使用方法

上面说了这些只是一个大致的介绍,我们还是来看看应该怎样使用吧。

你可以使用delegate的方式或者block的方式对view和viewModel进行桥接,在这里我们选择使用delegate,我认为这样看着比较直观,在代码中也更加明确。

首先创建一个工程,选择singleViewApplication,我们就以最常见的的登陆功能作为示例

### 新建用于由viewModel调用,view进行响应的protocol

首先要起个名字,就叫LoginViewModelDelegateProtocol 吧

```objectivec
@protocol LoginViewModelDelegateProtocol <NSObject>

@end
```

好,让我们想一想view会发送一些什么数据给VM ，VM都需要什么数据。

对于简单登陆的VM来说,我们需要通知view的数据和方法

- 登陆成功
- 错误提示
- 按钮状态改变（是否可以点击）

那么我们可以在protocol中添加方法了 

```objectivec
@protocol LoginViewModelDelegate <NSObject>

- (void)loginSuccess;
- (void)showTips:(NSString *)tip;
- (void)buttonEnable:(BOOL )enable;

@end
```

### 新建用于由view调用,viewModel进行响应的protocol

同理,我们只要确认vm和v需要交换的数据就好了。

- 用户名输入框的字符串
- 密码输入框的字符串
- 点击登陆事件

```objectivec
@protocol LoginViewModelInterface <NSObject>

- (void)inputUserName:(NSString *)uname;
- (void)inputPwd:(NSString *)pwd;
- (void)didTapLoginBUtton;

@end
```

### 实现代理方法

下面我们新建一个viewModel叫做LoginViewModel

LoginViewModel.h

```objectivec
#import <Foundation/Foundation.h>
#import "LoginViewModelDelegate.h"
#import "LoginViewModelInterface.h"

NS_ASSUME_NONNULL_BEGIN

@interface LoginViewModel : NSObject <LoginViewModelInterface>

@property (nonatomic, weak) id<LoginViewModelDelegate> delegate;

@end

NS_ASSUME_NONNULL_END
```

LoginViewModel.m

```objectivec
#import "LoginViewModelDelegateProtocol.h"

@interface LoginViewModel ()

@property (assign, nonatomic) BOOL unameValid;

@property (assign, nonatomic) BOOL pwdValid;

@end

@implementation LoginViewModel

- (void)inputUserName:(NSString *)uname {
    self.unameValid = uname.length>0;
    [self judgeAllValid];
}
- (void)inputPwd:(NSString *)pwd {
    self.pwdValid = pwd.length>0;
    [self judgeAllValid];
}
- (void)didTapLoginBUtton {
    // 一些请求,这里忽略网络请求,直接模拟结果
    [self.delegate loginSuccess];
}

- (void)judgeAllValid {
    BOOL v = [self isAllValid];
    [self.delegate buttonEnable:v];
}

- (BOOL)isAllValid {
    return self.unameValid && self.pwdValid;
}

@end
```

然后在controller初始化,并且实现全部的方法就可以了。

viewController.m

```objectivec
#import "ViewController.h"
#import "LoginViewModelDelegate.h"
#import "LoginViewModel.h"

@interface ViewController () <LoginViewModelDelegate, UITextFieldDelegate>
@property (nonatomic ,strong) LoginViewModel *vm;
@property (weak, nonatomic) IBOutlet UITextField *unameTxf;
@property (weak, nonatomic) IBOutlet UITextField *pwdTxf;
@property (weak, nonatomic) IBOutlet UIButton *loginBtn;

@end

@implementation ViewController

- (void)viewDidLoad {
    
    [super viewDidLoad];
    self.vm.delegate = self;
}

- (IBAction)toggleLogin:(id)sender {
    [self.vm didTapLoginBUtton];
}

#pragma mark - UITextFieldDelegate

- (BOOL)textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string {
    
    NSString *str = [textField.text stringByReplacingCharactersInRange:range withString:string];
    
    switch (textField.tag) {
        case 0:
            [self.vm inputUserName:str];
            break;
            
        case 1:
            [self.vm inputPwd:str];
            break;
    }
    return true;
}

#pragma mark - VMDelegate

- (void)loginSuccess {
    
    [self showTips:@"Login Success"];
    
}
- (void)showTips:(NSString *)tip {
    UIAlertController *alertVC = [UIAlertController alertControllerWithTitle:@"Tip" message:tip preferredStyle:UIAlertControllerStyleAlert];
    UIAlertAction *action = [UIAlertAction actionWithTitle:@"cancel" style:UIAlertActionStyleCancel handler:nil];
    [alertVC addAction:action];
    
    [self presentViewController:alertVC animated:true completion:nil];
}
- (void)buttonEnable:(BOOL )enable {
    self.loginBtn.enabled = enable;
}

#pragma mark - Getter

- (LoginViewModel *)vm {
    if (!_vm) {
        _vm = [LoginViewModel new];
    }
    return _vm;
}

@end
```

这里全部的代码是我手写的,后面省略了一些UIKit相关的布局,想必以你的聪明才智应该已经能很轻松的将剩余的补全了吧。

## 注意事项

这里有几点我认为应该注意的:

- vm与v之间不论通过什么传递值和响应,都要保持数据的单向流动。
- 代理用weak,内存回收时会自动置为nil,
- 全部model与viewModel中不应包含任何UIKit框架下的类

## 总结

总而言之,我认为MVVM在我们的代码整体分工和应用架构的过程中应用还是十分优雅的,

不过话说回来什么架构也罢，还是要看什么场景的，脱离了场景说这些都是扯淡不是吗