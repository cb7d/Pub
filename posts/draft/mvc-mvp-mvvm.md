# MV-X

架构模式小议

其实架构的区别很简单，不明白为什么很多文章说的那么复杂

## MVC

- Model 模型层
- View 视图层
- Controller 控制层

### 通信方式

单向环路通信 m -> v -> c -> m

模型层的变动直接通知视图层，视图的变动发送给控制器，控制器接受到变动通知模型层

## MVP

- Model 模型层
- View 视图层
- Presenter 表现层

### 通信方式

以Presenter为中心，其他层仅与P层通信并且是双向通信

m <-> p <-> v

视图和表现层双向通信，模型层和表现层双向通信

## MVVM

- Model 模型层

- View 视图层

- ViewModel 视图模型层

  把Presenter换为ViewModel，MVVM 基本上与 MVP 是一样的，区别就在于MVVM中采用ViewModel与View的双向绑定，视图与视图模型的变化会互相反馈

