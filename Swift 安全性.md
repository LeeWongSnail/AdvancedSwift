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

