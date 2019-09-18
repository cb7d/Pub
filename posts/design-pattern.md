# Design pattern

怎样写出优雅，直观，清晰的代码？这就少不了对设计模式的理解与应用，这篇主要总结归纳一下常见的设计模式和在iOS开发过程中的应用

在这之前，我们需要先回顾一下设计原则

- 单一职责原则 (Single Responsibility Principle)
- 开放-封闭原则 (Open-Closed Principle)
- 里氏替换原则 (Liskov Substitution Principle)
- 依赖倒转原则 (Dependence Inversion Principle)
- 接口隔离原则 (Interface Segregation Principle)
- 迪米特法则 (Law Of Demeter)
- 组合/聚合复用原则 (Composite/Aggregate Reuse Principle)

下面是不同设计模式的分类，大致分为三类：

- [创建型模式](#创建型模式)
- [结构型模式](#结构型模式)
- [行为型模式](#行为型模式)

## 创建型模式

- [简单工厂模式](#简单工厂模式)
- [工厂模式](#工厂模式)
- [抽象工厂模式](#抽象工厂模式)
- [单例模式](#单例模式)
- [建造者模式](#建造者模式)
- [原型模式](#原型模式)

### 简单工厂模式

描述：简单工厂模式(Simple Factory Pattern)：又称为静态工厂方法(Static Factory Method)模式，它属于类创建型模式。在简单工厂模式中，可以根据参数的不同返回不同类的实例。简单工厂模式专门定义一个类来负责创建其他类的实例，被创建的实例通常都具有共同的父类。

假设你有一个商店，在卖两种产品

```swift
protocol Product {}

class Product_A: Product {}
class Product_B: Product {}


class Store {
    func sellProduct(type: Int) -> Product? {
        if type == 0 {
            print("sell Product_A")
            return Product_A()
        } else if type == 1 {
            print("sell Product_B")
            return Product_B()
        }
        return nil;
    }
}
let store = Store()
Store.sellProduct(type: 0)
Store.sellProduct(type: 1)

```

这其实是无工厂模式，直接在方法里面判断最终要卖出的产品，而工厂方法如下

```objectivec
protocol Product {}

class Product_A: Product {}
class Product_B: Product {}

class Factory {
    func sellProduct(type: Int) -> Product? {
        if type == 0 {
            print("sell Product_A")
            return Product_A()
        } else if type == 1 {
            print("sell Product_B")
            return Product_B()
        }
        return nil;
    }
}

class Store {
    let factory = Factory()
}

let store = Store()

store.factory.sellProduct(type: 0)
store.factory.sellProduct(type: 1)
```

现在，产品直接由工厂进行接管，我们的店铺不在强依赖产品，只需要提供工厂就好了。

优点：

- 调用者依赖工厂，可以免除对具体产品的直接依赖
- 调用者无需知道具体产品，直接传递对应的参数即可

缺点：

- 工厂集中了产品逻辑，一旦工厂无法正常工作整个系统将会受到影响
- 添加新产品将修改工厂的逻辑，违反开闭原则

### 工厂模式

描述：定义一个接口用于创建对象，但是让子类决定初始化哪个类。工厂方法把一个类的初始化下放到子类。

```swift
protocol Product {}

class Product_A: Product {}
class Product_B: Product {}

class Factory {
    // 工厂类负责定义接口
    func sellProduct() -> Product? { return nil }
    
    func sellProduct(type: Int) -> Product? {
      	// 产品的实例化下放到工厂的子类中进行，有工厂子类负责
        if type == 0 {
            return FactoryA().sellProduct()
        } else if type == 1 {
            return FactoryB().sellProduct()
        }
        return nil;
    }
}

class FactoryA: Factory {
    override func sellProduct() -> Product? {
        print("sell Product_A")
        return Product_A();
    }
}

class FactoryB: Factory {
    override func sellProduct() -> Product? {
        print("sell Product_B")
        return Product_B();
    }
}

class Store {
    let factory = Factory()
}

let store = Store()

store.factory.sellProduct(type: 0)
store.factory.sellProduct(type: 1)
```

相较于简单工厂模式，工厂模式定义了工厂的子类用于创建产品。

优点：

- 用户只需要关心所需产品对应的工厂，无须关心创建细节，甚至无须知道具体产品类的类名。
- 工厂可以完全自主决定创建产品的细节，实现逻辑由创造哪一种产品变为了选择哪一个共产来生产，职责抽象了出来

缺点：

- 增加了抽象层会给理解带来一些难度，同时在实现时可能需要使用反射等技术，增加了实现的复杂度
- 在添加新产品时还需为产品添加新的工厂，增加了复杂度

### 抽象工厂模式

描述：为一个产品族提供了统一的创建接口。当需要这个产品族的某一系列的时候，可以从抽象工厂中选出相应的系列创建一个具体的工厂类。

也即是说，之前我们的工厂对应的产品线是单一的，现在如果我们的工厂扩展了一系列的产品，就需要使用抽象工厂模式

```swift
protocol Product {}

class Product_A_1: Product {}
class Product_A_2: Product {}
class Product_B_1: Product {}
class Product_B_2: Product {}

class Factory {
    // 贩卖1系列产品
    func sellProduct1() -> Product? { return nil }
  	// 贩卖2系列产品
    func sellProduct2() -> Product? { return nil }
    
    func sellProductA(type: Int) -> Product? {
        if type == 0 {
            return FactoryA().sellProduct1()
        } else if type == 1 {
            return FactoryA().sellProduct2()
        }
        return nil;
    }
    
    func sellProductB(type: Int) -> Product? {
        if type == 0 {
            return FactoryB().sellProduct1()
        } else if type == 1 {
            return FactoryB().sellProduct2()
        }
        return nil;
    }
}

class FactoryA: Factory {
    override func sellProduct1() -> Product? {
        return Product_A_1()
    }
    
    override func sellProduct2() -> Product? {
        return Product_A_2()
    }
}

class FactoryB: Factory {
    override func sellProduct1() -> Product? {
        return Product_B_1()
    }
    
    override func sellProduct2() -> Product? {
        return Product_B_1()
    }
}

class Store {
    let factory = Factory()
}

let store = Store()

store.factory.sellProductA(type: 0)
store.factory.sellProductA(type: 1)
store.factory.sellProductB(type: 0)
store.factory.sellProductB(type: 1)
```

优点：

- 抽象工厂隔离了对应真实工厂类的生成，这使得更换具体工厂变得相对容易
- 能够保证调用者只使用同一工厂生产的产品

缺点：

- 添加新产品型号时修改将会设计到抽象工厂以及其全部子类，比较麻烦
- 开闭原则的倾斜性（增加新的工厂和产品族容易，增加新的产品等级结构麻烦）

### 单例模式

描述：确保一个类只有一个实例，并提供对该实例的全局访问。

```swift
class Singleton {
    static let instance = Singleton()
    private init() {
        
    }
}

let singleton = Singleton.instance
```

这里没什么可多说的，在系统中很多地方都需要全局中使用同一个实例，如 `UIApplication.shared`，

单例模式的使用有三个要素

- 这个类只能有一个实例
- 这个实例必须是该类自行创建
- 它必须自行向整个系统提供该实例

优点：

- 提供了共享数据的方法
- 避免频繁创建销毁所带来的开销

缺点：

- 单例对象职责过重，某种程度破坏了单一职责原则
- 滥用单例会导致在有垃圾自动回收功能的系统中持续占用内存，单例模式应用的数量应该是越少越好

### 建造者模式

描述：建造者模式（Builder Pattern）也叫做生成器模式，使用多个简单的对象一步一步构建成一个复杂的对象。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。一个 Builder 类会一步一步构造最终的对象。该 Builder 类是独立于其他对象的

```swift
class Product {
    var parts: String = ""
}

protocol Builder {
    
    func buildPart()
    
    func getProduct() -> Product
}

class Director {
    let concreteBuilder: Builder
    init(concreteBuilder: Builder) {
        self.concreteBuilder = concreteBuilder
    }
    
    func construct() ->Product{
        concreteBuilder.buildPart()
        return concreteBuilder.getProduct()
    }
}

class ConcreteBuilder: Builder{
    
    var product = Product()
    
    func buildPart() {
        product.parts = "Finish"
    }
    
    func getProduct() -> Product{
        return product
    }
}

let director = Director(concreteBuilder: ConcreteBuilder())
let product = director.construct()
print(product.parts)
```

主要场景为当构建一个复杂的产品的时候，往往产品拥有很多的组成部分，最终用户无须知晓产品构建的具体细节，这时可以使用建造者模式对其进行设计与描述

优点：

- 产品本身与产品的创建过程解耦
- 方便更换新的建造者，用户可以使用不同的建造者构造新的产品
- 更加精细的控制产品的创建过程
- 增加建造者无需修改原类库的代码，方便扩展，符合开闭原则

缺点：

- 产品之前差异过大不适用
- 产品内部变化复杂可能需要增加很多建造者，导致系统变得庞大

### 原型模式

描述：用原型实例指定创建对象的种类，并且通过拷贝这些原型,创建新的对象。

```swift
protocol Prototype {
    func clone() -> Prototype
}

struct Product: Prototype {
    let name: String
    func clone() -> Prototype {
        return Product(name: name)
    }
}

let product = Product(name: "apple")
let copy = product.clone()
```

大多数时候，你可以将原型模式当作深拷贝来理解，用处就是像细胞分裂一样生成大量同样内容的对象

## 结构型模式

- [策略模式](#策略模式)
- [模版模式](#模版模式)
- [观察者模式](#观察者模式)
- [迭代器模式](#迭代器模式)
- [责任链模式](#责任链模式)
- [命令模式](#命令模式)
- [备忘录模式](#备忘录模式)
- [状态模式](#状态模式)
- [访问者模式](#访问者模式)
- [中介者模式](#中介者模式)

### 策略模式

### 模版模式

### 观察者模式

### 迭代器模式

### 责任链模式

### 命令模式

### 备忘录模式

### 状态模式

### 访问者模式

### 中介者模式

## 行为型模式

- [适配器模式](#适配器模式)
- [装饰器模式](#装饰器模式)
- [代理模式](#代理模式)
- [外观模式](#外观模式)
- [桥接模式](#桥接模式)
- [组合模式](#组合模式)
- [享元模式](#享元模式)

### 适配器模式

### 装饰器模式

### 代理模式

### 外观模式

### 桥接模式

### 组合模式

### 享元模式

参考

https://juejin.im/post/5b827f0df265da43412875dd#heading-11

[https://zh.wikipedia.org/wiki/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F_(%E8%AE%A1%E7%AE%97%E6%9C%BA)](https://zh.wikipedia.org/wiki/设计模式_(计算机))

https://cloud.tencent.com/developer/article/1412368

[http://www.licheng244.com/posts/06%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/23%E7%A7%8D%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.html](http://www.licheng244.com/posts/06设计模式/23种设计模式.html)