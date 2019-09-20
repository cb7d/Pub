# atomic

## 前言

在使用 `@property` 修饰成员变量的时候，会设置一个默认的缺省值 **atomic**

```swift
@property (assign) NSInteger ticketNum;
```

以上写法等同于

```swift
@property (atomic, assign) NSInteger ticketNum;
```

















## 安全性证明

## 实现源码

## 总结

