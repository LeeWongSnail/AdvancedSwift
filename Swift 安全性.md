## Swift 安全性

### 代码安全

#### let属性

 使用`let`申明常量避免被修改。

#### 值类型

值类型可以避免在方法调用等参数传递过程中状态被修改。

与对象类型对应，对象类型在方法传递过程当中是指针传递，值类型即对应的`struct`,当作为参数传递而被修改时不会修改对象本身

#### 访问控制

访问控制有点复杂这里我们单独写一篇文章来描述

[访问控制](https://github.com/LeeWongSnail/AdvancedSwift/blob/main/Swift-%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6.md)

#### 强制异常处理

方法需要抛出异常时，需要申明为throw方法。当调用可能会throw异常的方法，`需要强制捕获异常避免将异常暴露到上层`

#### 模式匹配

通过模式匹配检测Switch中未处理的case

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7feznzfdjj310605tjrr.jpg)

### 类型安全

#### 强制类型转换

禁止隐式类型转换，避免转换中带来的异常问题。同时类型转换不会带来额外的运行时消耗。

这样就可以避免我们在OC中，经常遇到因为类型不匹配导致的找不到方法crash

#### keyPath

KeyPath相比使用字符串可以提供属性名和类型信息，可以利用编译器检查。

keyPath更详细的介绍请看[这里](https://github.com/LeeWongSnail/AdvancedSwift/blob/main/KeyPath.md)	
#### 泛型

提供泛型和协议关联类型，可以编写出类型安全的代码。相比Any可以更多利用编译时检查发现类型问题。

我们这里也用一个单独的篇幅来了解[泛型](https://github.com/LeeWongSnail/AdvancedSwift/blob/main/%E6%B3%9B%E5%9E%8B.md)

#### Enum关联类型

通过给特定枚举指定类型避免使用Any


#### Enum关联类型

通过给特定枚举指定类型避免使用Any

更多关于enum的知识点请看[这里](https://github.com/LeeWongSnail/AdvancedSwift/blob/main/Swift%20Enum.md)


### 内存安全

#### 空安全 

通过标识可选值避免空指针带来的异常问题，这里引入的是[可选型](https://github.com/LeeWongSnail/AdvancedSwift/blob/main/%E5%8F%AF%E9%80%89%E5%9E%8B%20Optional.md)，这是在OC中所没有的，简单一点来说可选型即可能为空的。

#### ARC

使用自动内存管理避免手动管理内存带来的各种内存问题

#### 强制初始化

变量使用前必须初始化，不同于OC，Swift的变量在使用前都必须初始化

- 类中的let定义的常量 要么定义时就初始化完成，要么在类的init方法中初始化否则会直接报错

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7k1nouu8wj314q072aau.jpg)

- 类型定义的可选型变量 默认为nil 默认已经进行了初始化

```swift
var name: String?
var name1: String? = Optional.none
```

- 使用!定义的变量标识这个变量肯定有值(在变量被使用之前肯定会被赋值)

```swift
var sName: String! = "Lee"
```

- 懒加载，在变量被使用到的时候被赋值

```swift
private lazy var tiger: Tiger = {
    let tiger = Tiger()
    return tiger
}()
```

#### 内存独占访问

swift的安全性这里也单独写了篇文章来描述[内存安全](https://github.com/LeeWongSnail/AdvancedSwift/blob/main/%E5%9F%BA%E7%A1%80/%E5%86%85%E5%AD%98%E5%AE%89%E5%85%A8.md)


