### Method Dispatch



### 简介

`method Dispatch`即方法派发，是指我们方法的调用过程，我们如何根据方法名找到对应的执行进行执行。

先来看下原始的英文定义
```
As definition says, Method Dispatch is how a program selects which instructions to execute when invoking a method.
It’s something that happens every time a method is called, and not something that you tend to think a lot about.
```

方法派发主要有下面三种类型：

- STATIC DISPATCH(aka compile-time dispatch aka direct dispatch)
- TABLE DISPATCH(aka Runtime dispatch aka Dynamic dispatch)
- MESSAGE DISPATCH

下面我们来介绍下这几种方法的派发方式

### STATIC DISPATCH

译为`静态派发`或`直接派发`，这是最快的一种派发方式，不进产生更少的汇编代码，也允许编译器做各种各样的优化，比如 `inline code`。

但是，静态派发从编码的角度来说也有很多限制，例如他的动态性不能支持继承(subClassing)。

`注意`: extension中的方法都是静态派发，值类型(struct)为静态派发，使用static和final修饰的引用类型也是静态派发


### TABLE DISPATCH

译为:`表派发`，是最常见的动态性实现方式。表派发在每个类中使用一个包含此类所有方法的函数指针数组。通常称之为`virtual table（虚表函数）`.每个子类都保存一份函数表的拷贝，只是被重写的方法入口函数指针不同。当子类添加了新的方法时，新的方法会被添加到数组的末尾。在运行时，会查询这个表来决定具体要执行哪一个方法。

![tabledispatch_method](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/tabledispatch.jpeg)

![tabledispatch](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/tabledispatch_method.png)

查询表的操作很简单，但是相比于静态派发会多进行两次读取和一次跳转，增加了一些开销。尤其是这里编译器无法像静态派发一样无法基于发生在函数内部的操作进行任何优化

这种方式有一个弊端:extension无法扩充派发表。因为子类会向派发表末尾添加新的方法，所以，对于扩展来说，没有可以安全的向派发表添加函数指针的索引。

### Message Dispatch

译为：`消息派发`，这是动态性最强的派发方法，但是也是最慢的派发方法。在table Dispatch中witness table在编译时被生成，但是消息派发无法在编译是确定具体应该调用哪个方法，因为他可能被方法交换，又或者运行时动态添加的新方法。但他确实Cocoa矿建开发的基础，实现了诸如`KVO`、`CoreData`等底层机制。

当消息被派发时，运行时会缓慢的查找类的继承树来确定应该执行哪一个方法，实际上在添加了高性能的方法缓存偶这个过程并没有很慢。

编译器会一直尝试升级派发方式到静态派发，除非指定`dynamic`或`@objc`关键词

![](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/messagedispatch.png)

### 示例

我们先来看下 下面这部分代码

```swift
class NewViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
    }
}

class ParentClass: NSObject {
    func sayHi() {
        print("ParentClass - sayHi")
    }
}

func hello(person: ParentClass) {
    person.sayHi()
}

class ChildClass: ParentClass {
}

extension ChildClass {
    override func sayHi() {
        print("ChildClass - sayHi")
    }
}
```
编译时我们发现

![sayhierror](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/sayhierror.png)

`Non-@objc instance method 'sayHi()' declared in 'ParentClass' cannot be overridden from extension`

编译器提示在ParentClass中没有@objc的实例方法'sayHi()'的声明，因此不能在extension中重写。

这就意味着编译器没有在ParentClass中找到'sayHi()'。但是很明显，我们的代码中是有的，所以只能表明编译器没有找到。我们先尝试根据错误提示去修复这个错误。

首先想到，提示中的`@objc`，是否我们在方法前添加了`@objc`就可以了呢？

```swift
class ParentClass: NSObject {
    @objc func sayHi() {
        print("ParentClass - sayHi")
    }
}
```
结果很明显还是不行，不过提示变了变为`Cannot override a non-dynamic class declaration from an extension`。

我们继续在方法前添加`dynamic`修饰:

```swift
class ParentClass: NSObject {
    @objc dynamic func sayHi() {
        print("ParentClass - sayHi")
    }
}
```

这下终于编译成功了。总结我们上面的修改，在方法前添加了`dynamic`和`@objc`两个关键词修饰。

这两个关键词分别代表什么意思呢？

`@objc`,`dynamic`: 表示这个方法的派发方式改为消息派发而不是表派发。

也就是说只有是消息派发的方式声明的方法才能在`extension`中`重载`，其他方法均无法在`extension`中`重载`。这也说明了对`Class`类型在extension中方法的派发方式是`消息派发`。对于其他类型 例如struct则依然是静态派发。

![messagedispatch_type](https://github.com/LeeWongSnail/AdvancedSwift/blob/main/res/messagedispatch_type.png)

`注意`:

- 值类型中的方法调用都是静态派发
- 类和协议的extension中的方法为静态派发
- NSObject的extension是消息派发
- NSObject在initial方法中声明的方法使用表派发
- 初始协议声明中方法(非extension中的方法)的默认实现使用表派


### 引用类型

```swift
protocol MyProtocol {
}

struct MyStruct: MyProtocol {
}

extension MyStruct {
    func extensionMethod() {
        print("In Struct")
    }
}

extension MyProtocol {
    func extensionMethod() {
        print("In Protocol")
    }
}

let myStruct = MyStruct()
let proto: MyProtocol = myStruct

myStruct.extensionMethod() // -> “In Struct”
proto.extensionMethod() // -> “In Protocol”
```

其实在没执行前我们预期两个方法调用的结果都是输出`In Struct`。但实际上确是输出了两个不同的结果。

对于`myStruct.extensionMethod()`的输出结果我们没有疑问，那对于`proto.extensionMethod() `呢？

首先我们确认在MyProtocol中extensionMethod是在extension实现的 根据上面的总结对于协议的extension，方法的派发方式是静态派发。也就是说会直接调用指定类型的extensionMethod方法，这时候proto对象就是一个MyProtocol类型的，所以这里会调用protocol方法。

那如果我们希望调用到`In Struct`这个结果呢？根据上面总结的，在协议中初始声明中的方法使用表派发

```
protocol MyProtocol {
    func extensionMethod()
}
```
则输出结果变为两个都是`In Struct`。

#### 指定派发方式

##### final

`final`使类中定义的方法采用`直接派发`。这个关键字移除了`任何动态行为`的可能性。它可以用来修饰方法，甚至是在本来已经采用直接派发的扩展中也可以。它还可屏蔽来自Objective-C runtime运行时的所有方法，因此也不会产生任何的selector（方法选择器）

##### dynamic

`dynamic`使类中定义的方法采用`消息派发`。它还会将方法暴露给`Objective-C runtime`运行时。`dynamic`可以用来使`扩展中声明的方法能够被重载`。dynamic关键字同时适用于NSObject子类和Swift类



### 参考文档

[Method Dispatch in Swift, and its effect on performance](https://medium.com/@venki0119/method-dispatch-in-swift-effects-of-it-on-performance-b5f120e497d3)

[Method Dispatch in Swift](https://medium.com/@pallavidipke07/method-dispatch-in-swift-b113a40a713a)

[Method Dispatch in Swift](https://www.rightpoint.com/rplabs/switch-method-dispatch-table)

