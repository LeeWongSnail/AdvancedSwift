# Swift class&static

## 适用场景

### static 适用场景

- 修饰存储属性
- 修饰计算属性
- 修饰类型方法

#### 当static修饰局部变量

当用static修饰局部变量时,局部变量的内存地址会从栈变为全局区(静态区),只在函数内部可见,只初始化一次,所以也只有一个内存地址。

示例:

```swift
class Person {
    static let id = 12345
    static var name: String = "LeeWong"
    static var height: CGFloat {
        return 2
    }
    static func sayHello() {}
}
```

#### 当static修饰全局变量

仍然是在静态储存区没变,生命周期为整个程序运行期间.在整个声明它的文件中可用,在声明他之外的文件之外不可见。

static修饰的变量即使所在的页面被销毁，变量值依然可以保存，因为是保存在全局区。因此有时候也可以用来做单次启动期间的一些值的记录。


### class适用场景

- 修饰类方法
- 修饰计算属性

示例:

```swift
class ClassDemo {
    class var height: CGFloat {
        return 25
    }
    
    class func sayHello() {}
}
```


##区别

### class不可以修饰类的存储属性，static可以修饰类的存储属性

![classstoreproperty]()

这时候编译器会提示`Class stored properties not supported in classes; did you mean 'static'?`

为什么class不可以修饰存储变量呢？

其实在OC中就没有类变量这个概念，因为类我们一般都需要通过创建实例来使用,而且我们在OC中了解到类型其实是元类的类对象，但是元类的类对象一般都不是可变的。

### protocol中使用static来修饰类型类型或者计算属性

在协议中我们是否可以使用class修饰protocol中的方法或者属性呢

```swift
protocol PersonProtocol {
    class func sayHi()
    class var name: String {get}
}
```
![classmethodinprotocol]()

显然class是不行的。

我们可以用static来修饰:

```swift
protocol PersonProtocol {
    static func sayHi()
    static var name: String {get}
}
```

无论是class还是struct都可以遵守这个协议

```swift
class Father: PersonProtocol {
    static func sayHi() {
        
    }
    
    static var name: String {
        return "name"
    }
}

struct Son: PersonProtocol {
    static func sayHi() {
        
    }
    
    static var name: String {
        return "LeeWong"
    }
}
```

### static 修饰的类方法不能继承，class修饰的类方法可以继承

class 是 `dynamically dispatched`，也就是在 `runtime` 才被指派，`compiler` 需要去找到底是 `class` 或是 `subclass` 的 `method`；換句話說就是用 `class` 可以被 `override`。反之就是 `statically dispatched`，在 `compile time` 就決定好了，運行效率較高。

使用static修饰会直接报错:

![overridestaticmethod]()

但是使用class就没有问题

```swift
class Father {
    class func sayHi() {
        
    }
}

class Son: Father {
    override class func sayHi() {
        
    }
}
```

`Static Dispatch is 4 times faster than Dynamic Dispatch as far as Objective-C runtime is used. This is true for swift too. However, it only takes 4 nanoseconds to dispatch Dynamically.`

### 修饰范围

`class`和`static`能够修饰的范围不一样,`class`只能在`class`中修饰,而`static`可以不仅可以作用于`class`中,也可以在`enum`,和`struct`中使用.

![classInstruct]()
    

## 参考

[Swift 的 property 大統整](https://medium.com/@ji3g4kami/swift-%E7%9A%84-property-%E5%A4%A7%E7%B5%B1%E6%95%B4-be9f25dfdb5f)

[说说iOS中的常用的关键字static ,class(仅限Swift关键字)](https://www.codercto.com/a/42612.html)

[Swift: static func VS class func
](https://kelvas09.github.io/blog/posts/static-func-vs-class-func/)

