Off-Screen

GPU屏幕渲染有以下两种方式：**

- On-Screen Rendering
  意为当前屏幕渲染，指的是GPU的渲染操作是在当前用于显示的屏幕缓冲区中进行。
- Off-Screen Rendering
  意为离屏渲染，指的是GPU在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。

**特殊的离屏渲染：**

>  如果将不在GPU的当前屏幕缓冲区中进行的渲染都称为离屏渲染，那么就还有另一种特殊的“离屏渲染”方式： CPU渲染。
> 如果我们重写了drawRect方法，并且使用任何Core Graphics的技术进行了绘制操作，就涉及到了CPU渲染。整个渲染过程由CPU在App内 同步地
> 完成，渲染得到的bitmap最后再交由GPU用于显示。
> **备注：CoreGraphic通常是线程安全的，所以可以进行异步绘制，显示的时候再放回主线程，一个简单的异步绘制过程大致如下**

触发条件

- shouldRasterize（光栅化）
- masks（遮罩）
- shadows（阴影）
- edge antialiasing（抗锯齿）
- group opacity（不透明）
- 复杂形状设置圆角等
- 渐变