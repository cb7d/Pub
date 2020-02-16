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

- [模版模式](#模版模式)
- [适配器模式](#适配器模式)
- [桥接模式](#桥接模式)
- [装饰模式](#装饰模式)
- [外观模式](#外观模式)
- [享元模式](#享元模式)
- [代理模式](#代理模式)

### 模版模式

描述：模版模式将常见的方法抽象出来，每个成员单独去实现模版中的方法，大多语言的实现可以通过虚基类来实现，在Swift中我们可以使用协议来完成

```swift
protocol Create {
    func createProduction()
}

class Factory: Create {
    func createProduction() {
        print("all production created")
    }
}

class Worker: Create {
    func createProduction() {
        print("product created")
    }
}

Worker().createProduction()

Factory().createProduction()
```

对于`createProduction`这个方法就是如此，成员只需遵守协议，具体实现内部自己处理

优点：

- 将不变的行为封装起来，避免重复定义
- 方便更改和扩展具体的实现

缺点：

- 抽象容易导致歧义，子类有可能不完全符合模版的方法声明



### 适配器模式

描述：将一个类的接口转化为使用者希望的接口，适配器模式使得原本无法共同工作的类能够一起工作

```swift
protocol Greeting {
    func sayHI()
}

struct Person: Greeting {
    func sayHI() {
        print("HI")
    }
}

struct Dog {
    func bark() {
        print("BarkBark")
    }
}

let person1 = Person()
let person2 = Person()

let dog1 = Dog()

person1.sayHI()
person2.sayHI()
dog1.bark()
```

这时我们希望dog也能使用同样的sayHI方法，这里我们可以使用Swift的extension来完成适配器模式

```swift
let dog1 = Dog()

extension Dog: Greeting {
    func sayHI() {
        bark()
    }
}

person1.sayHI()
person2.sayHI()
dog1.sayHI())
```

优点：

- 通过适配器重用原方法，无需改动原类
- 增加了类的透明性和适用性
- 灵活，可扩展，符合开闭原则

缺点：

- 对于无多继承和extension这种东西的语言实现起来较复杂

### 桥接模式

描述：将抽象部分与实现部分分离，使之都能独立的变化，又称作接口模式

```swift
protocol Action {
    func happened()
}

protocol Animal {
    var action: Action? { get }
    func walk()
}


struct People: Animal {
    var action: Action?
    func walk() {
        print("Person walk")
        action?.happened()
    }
}

struct Jump: Action {
    func happened() {
        print("Jump")
    }
}

let person = People(action: Jump())
person.walk()
```

可以看到，人在走路的时候会发生action，action在内部是抽象的，由外部控制

优点：

- 分离接口抽象以及实现部分
- 替代多继承的一种解决方案
- 桥接模式提高了系统的可扩展性
- 可以对调用方隐藏实现细节

缺点：

- 关联关系建立在抽象层，增加了系统的设计难度

### 装饰模式

描述：动态地给一个对象添加一些额外的职责。就增加功能来说，装饰器模式相比生成子类更为灵活。

```swift
protocol Component {
    func method() -> String
}

protocol Decorator: Component {
    var component: Component { get }
}

struct Coffee: Component {
    func method() -> String {
        return "making coffee"
    }
}

struct Sugar: Decorator {
    func method() -> String {
        return component.method() + " " + "add sugar"
    }
    
    var component: Component
}

struct Milk: Decorator {
    func method() -> String {
        return component.method() + " " + "add milk"
    }
    
    var component: Component
}

print(Milk(component: Sugar(component: Coffee())).method())
```

这里我们给咖啡加料，在加料有很多的情况下我们使用装饰模式就会很方便，在进行功能扩展的时候比子类化要灵活许多。装饰器模式主要的原理就在于将需要扩展的对象进行包装，并实现与该对象相同的接口，并在将任务传递给被包装的对象之前加入自己的行为。

优点：

- 比子类化的方案要灵活
- 可以创建多种排列组合
- 组件类和装饰器类可以独立变化

缺点：

- 经过多层装饰的对象在排查错误的时候比较复杂

### 外观模式

描述：外部与一个子系统的通信必须通过一个统一的外观对象进行，为子系统中的一组接口提供一个一致的界面，外观模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。外观模式又称为门面模式，它是一种对象结构型模式。

```swift
protocol Component {
    func method() -> String
}

protocol Decorator: Component {
    var component: Component { get }
}

struct Coffee: Component {
    func method() -> String {
        return "making coffee"
    }
}

struct Sugar: Decorator {
    func method() -> String {
        return component.method() + " " + "add sugar"
    }
    
    var component: Component
}

struct Milk: Decorator {
    func method() -> String {
        return component.method() + " " + "add milk"
    }
    
    var component: Component
}

print(Milk(component: Sugar(component: Coffee())).method())

struct Store {
    func saleCoffee() -> Component {
        return Milk(component: Sugar(component: Coffee()))
    }
}

Store().saleCoffee().method()
```

外观模式主要提供一个接口，外部的使用者去调用接口返回结果而不必关心内部系统的实现，还是以咖啡为例，顾客去购买咖啡只需调用商店的卖出方法，内部如何实现交给子系统

优点：

- 对客户屏蔽内部系统，客户代码将变得简单
- 实现了松耦合的关系
- 降低了大型系统中的编译依赖性

缺点：

- 不能很好地限制客户使用子系统类，如果对客户访问子系统类做太多的限制则减少了可变性和灵活性。
- 在不引入抽象外观类的情况下，增加新的子系统可能需要修改外观类或客户端的源代码，违背了“开闭原则”。

### 享元模式

描述：享元模式（Flyweight Pattern）主要用于减少创建对象的数量，以减少内存占用和提高性能。这种类型的设计模式属于结构型模式，它提供了减少对象数量从而改善应用所需的对象结构的方式。

这里主要可以参考UITableViewCell的重用机制，Cell的展示会发生变化，但是屏幕上能够展示出来的Cell是有限个的，利用重用机制可以很好的减少内存开销

优点：

- 较少的内存占用
- 享元对象能够在不同环境中共享

缺点：

- 享元对象需要区分内外状态
- 享元对象的状态外部化会使得读取外部状态的运行时间变长

### 代理模式

描述：给某一个对象提供一个代 理，并由代理对象控制对原对象的引用。代理模式的英 文叫做Proxy或Surrogate，它是一种对象结构型模式

```swift
protocol Guarder {
    func guardTheEntrance()
}

class Dog: Guarder {
    func guardTheEntrance() {
        print("看家中...")
    }
}

class Person {
    
    var guarder: Guarder
    
    init(guarder: Guarder) {
        self.guarder = guarder
    }
    
    func leaveHome() {
        guarder.guardTheEntrance()
    }
}

Person(guarder: Dog()).leaveHome()
```

代理模式就是类本身需要实现某种功能，但是为了降低耦合度本身不去实现，而是委托给其他类去实现

优点：

- 代理模式能够协调调用者和被调用者，在一定程度上降低了系 统的耦合度。

缺点：

- 一定程度上加大了系统的内存开销

## 行为型模式

- [责任链模式](#责任链模式)
- [命令模式](#命令模式)
- [解释器模式](#解释器模式)
- [迭代器模式](#迭代器模式)
- [中介者模式](#中介者模式)
- [观察者模式](#观察者模式)
- [状态模式](#状态模式)
- [备忘录模式](#备忘录模式)
- [策略模式](#策略模式)

### 责任链模式

描述：避免请求发送者与接收者耦合在一起，让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止。

```swift
protocol Leader {
    
    var leader: Leader? { get }
    
    func accept() -> Bool
}

class Employee: Leader {
    var leader: Leader?
    
    init(leader: Leader?) {
        self.leader = leader
    }
    
    func accept() -> Bool {
        print("deal with thing")
        return true && leader?.accept() ?? true
    }
}

let leader1 = Employee(leader: nil)
let leader2 = Employee(leader: leader1)
let leader3 = Employee(leader: leader2)

leader3.accept()
```

比如你要向领导请假，需要一级一级的同意，如上。UIKit的touch事件就应用了责任链模式，使得点击事件能够不断传递。

优点：

- 使得客户端与具体响应者能够松耦合
- 方便响应者更新响应的方法

缺点：

- 由于存在响应链条，处理起来的资源占用可能增加

### 命令模式

描述：将一个请求封装成一个对象，从而使用户可以用不同的请求对客户进行参数化。

命令模式有两个特点：

- 命令逻辑化，命令的逻辑可以由外部传入或内部实现
- 支持撤销命令（undo）

```swift
protocol Command {
    var operation: () -> Void { get }
    var undoOperation: () -> Void { get }
}

struct WashCommand: Command {
    var operation: () -> Void
    var undoOperation: () -> Void
}

struct Person {
    var command: Command
    func execute() {
        command.operation()
    }
    func undo() {
        command.undoOperation()
    }
}

let washCommand = WashCommand(operation: {
    print("wash clothes")
}) {
    print("put clothed into sky")
}

let person = Person(command: washCommand)

person.execute()

person.undo()
```

优点：

- 适合组合命令来实现复杂功能

缺点：

- 每个命令都需要具体的类来执行，会创建大量的命令类

### 解释器模式

描述：给定一个语言，定义它的文法的一种表示，并定义一个解释器，这个解释器使用该表示来解释语言中的句子。

```swift
protocol Expression {
    func eval(context: String) -> Int
}

struct AddCalculator: Expression {
    func eval(context: String) -> Int {
        return context.components(separatedBy: "+").compactMap{ Int($0) }.reduce(0, +)
    }
}

AddCalculator().eval(context: "1+1")
```

优点：

- 解释器扩展性强

缺点：

- 在解释执行的时候往往会采用递归的形式，效率较低

### 迭代器模式

描述：提供一种方法顺序访问一个聚合对象中各个元素, 而又无须暴露该对象的内部表示。

```swift
struct Product {
    let id: Int
}

protocol Iterator {
    func hasNext() -> Bool
    func next() -> Product?
}

class Store: Iterator {
    
    var products: [Product]
    var index = 0;
    
    init(products: [Product]) {
        self.products = products
    }
    
    func hasNext() -> Bool {
        return products.count > index
    }
    func next() -> Product? {
        defer {
            index += 1
        }
        
        return products[index]
    }
}

let store = Store(products: Array(1...100_).map{ Product(id: $0) })

while store.hasNext() {
    print(store.next()!)
}
```

与责任链相比，迭代器更加强调一次完整的遍历，商店只要有货物就会去销售获取，而责任链是随时可以中断的，比较强调响应者的概念

优点：

- 简化了集合的遍历方式
- 可以为某一个集合提供多种便利方式

缺点：

- 对于简单的集合类型来说增加了复杂度

### 中介者模式

描述：用一个中介对象来封装一系列的对象交互，中介者使各对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。中介者模式又称为调停者模式，它是一种对象行为型模式。

```swift
class Colleague {
    let name: String
    let mediator: Mediator
    
    init(name: String, mediator: Mediator) {
        self.name = name
        self.mediator = mediator
    }
    
    /**
     发送消息
     
     - parameter message: 消息
     */
    func send(message: String) {
        print("Colleague \(name) send: \(message)")
        mediator.send(message: message, colleague: self)
    }
    
    /**
     接收消息
     
     - parameter message: 消息
     */
    func receive(message: String) {
        assert(false, "Method should be overriden")
    }
}

/**
 *  中介者接口
 */
protocol Mediator {
    /**
     发送消息
     
     - parameter message:   消息
     - parameter colleague: 发送者
     */
    func send(message: String, colleague: Colleague)
}

/// 具体中介者
class MessageMediator: Mediator {
    private var colleagues: [Colleague] = []
    
    func addColleague(colleague: Colleague) {
        colleagues.append(colleague)
    }
    
    func send(message: String, colleague: Colleague) {
        for c in colleagues {
            if c !== colleague { //for simplicity we compare object references
                c.receive(message: message)
            }
        }
    }
}

/// 具体对象
class ConcreteColleague: Colleague {
    override func receive(message: String) {
        print("Colleague \(name) received: \(message)")
    }
}

let messagesMediator = MessageMediator()
let c1 = ConcreteColleague(name: "A", mediator: messagesMediator)
let c2 = ConcreteColleague(name: "B", mediator: messagesMediator)
let c3 = ConcreteColleague(name: "C", mediator: messagesMediator)
messagesMediator.addColleague(colleague: c1)
messagesMediator.addColleague(colleague: c2)
messagesMediator.addColleague(colleague: c3)

c3.send(message: "Hello")
```

我们在工程中模块化解耦的方式就是使用了中介者模式。和观察者模式很像，区别在于观察者是不关心接受方的广播，中介者是介入两个（或多个）对象之间的定点消息传递。

优点：

- 模块化解耦
- 减少了子类的产生
- 简化了对象间的交互

缺点：

- 中介者本省容易变得臃肿

### 观察者模式

描述：定义对象间的一种一对多依赖关系，使得每当一个对象状态发生改变时，其相关依赖对象皆得到通知并被自动更新。观察者模式又叫做发布-订阅（Publish/Subscribe）模式、模型-视图（Model/View）模式、源-监听器（Source/Listener）模式或从属者（Dependents）模式。

```swift
protocol Observer {
    func receiveChanges()
    func hashValue() -> Int
}

extension Observer {
    
    func hashValue() -> Int {
        return Unmanaged<AnyObject>.passUnretained(self as AnyObject).toOpaque().hashValue
    }
}

protocol Observable {
    var observers: [Observer] { get }
    func add(observer: Observer)
    func remove(observer: Observer)
    func notify()
}

class ConcreteObserver: Observer {
    let name: String
    
    init(name: String) {
        self.name = name
    }
    func receiveChanges() {
        print("\(name) receive")
    }
}

class ConcreteObservable: Observable {
    
    var observers: [Observer] = []
    
    func add(observer: Observer) {
        observers.append(observer)
    }
    
    func remove(observer: Observer) {
        
        guard let idx = observers.firstIndex(where: { (obs) -> Bool in
            return obs.hashValue() == observer.hashValue()
        }) else { return  }
        
        observers.remove(at: idx)
    }
    
    func notify() {
        
        observers.forEach{
            $0.receiveChanges()
        }
    }
}

let c1 = ConcreteObserver(name: "A")
let c2 = ConcreteObserver(name: "B")
let c3 = ConcreteObserver(name: "C")

let observable = ConcreteObservable()
observable.add(observer: c1)
observable.add(observer: c2)
observable.add(observer: c3)

observable.notify()

observable.remove(observer: c1)
observable.notify()
```

在Cocoa框架中提供的[KVO](https://k.felixplus.top/kvo/)和通知中心就是观察者模式的实现。

优点：

- 可以实现表示层和数据逻辑层分离
- 支持广播通信

缺点：

- 将所有观察者都通知到会花费一些时间
- 需要注意循环依赖的事件发生

### 状态模式

描述：允许对象在内部状态发生改变时改变它的行为，对象看起来好像修改了它的类。

```swift
protocol State {
    func doStuff()
}

class StateA: State {
    func doStuff() {
        print("do some A stuff")
    }
}

class StateB: State {
    func doStuff() {
        print("do some B stuff")
    }
}

class Context {
    var state: State = StateA()
    func operation() {
        state.doStuff()
    }
}

let ctx = Context()

ctx.operation()

ctx.state = StateB()

ctx.operation()
```

优点：

- 只需改变状态就可以改变类的行为
- 允许状态转换对象与状态对象合为一体
- 多个对象可以共享状态

缺点：

- 违背开闭原则
- 使用不当会使得代码结构混乱

### 备忘录模式

描述：在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态。

```swift
class Todo: Codable{
    
    var content = ""
}

class Memento {
    func save(todo: Todo, key: String) {
        do {
            let data = try JSONEncoder().encode(todo)
            print(data)
            UserDefaults.standard.set(data, forKey: key)
            UserDefaults.standard.synchronize()
        } catch {
            print(error)
        }
    }
    
    func load(key: String) -> Todo? {
        
        guard let data = UserDefaults.standard.data(forKey: key) else { return nil }
        
        do {
            
            let todo = try JSONDecoder().decode(Todo.self, from: data)
            return todo
        } catch  {
            print(error)
        }
        return nil
    }
}

let todo = Todo()
let memento = Memento()

todo.content = "a simple thing"
memento.save(todo: todo, key: "key")

if let todo1 = memento.load(key: "key") {
    print("\(todo) -- \(todo1)")
}
```

备忘录模式可以通俗的理解为游戏的存档和读档。

优点：

- 提供给了用户可以保存状态的机制

缺点：

- 创建备忘会消耗资源

### 策略模式

描述：策略模式属于对象的行为模式，将某一组算法封装起来，让它们可以相互替换，策略模式提供了一种可插入式算法的实现方案

```swift
protocol Strategy {
    func saveData()
}

class MemoryStrategy: Strategy {
    func saveData() {
        print("save data to memory")
    }
}

class DiskStrategy: Strategy {
    func saveData() {
        print("save data to disk")
    }
}

class Downloader {
    let strategy: Strategy
    init(strategy: Strategy) {
        self.strategy = strategy
    }
    
    func download() {
        self.strategy.saveData()
    }
}

Downloader(strategy: MemoryStrategy()).download()
Downloader(strategy: DiskStrategy()).download()
```

如上，当我们调用下载方法时期望能够制定我们下载后的缓存策略

优点：

- 符合开闭原则，能够灵活的增加和修改实现
- 提供了算法复用的实现

缺点：

- 需要自定义策略支持
- 策略过多会在选择上花费一些功夫

## iOS常见的设计模式实现

### 单例模式

- UIApplication `sharedApplication`
- NSBundle `mainBundle`
- NSFileManager `defaultManager`
- NSNotificationCenter `defaultCenter`
- NSUserDefaults `standardUserDefaults`

### 代理模式

- UITableView
- UITextView
- UICollectionView
- UIScrollView
- UIGestureRecognizer

### 观察者模式

- Notification
- KVO

参考

https://juejin.im/post/5b827f0df265da43412875dd#heading-11

[https://zh.wikipedia.org/wiki/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F_(%E8%AE%A1%E7%AE%97%E6%9C%BA)](https://zh.wikipedia.org/wiki/设计模式_(计算机))

https://cloud.tencent.com/developer/article/1412368

[http://www.licheng244.com/posts/06%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/23%E7%A7%8D%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.html](http://www.licheng244.com/posts/06设计模式/23种设计模式.html)