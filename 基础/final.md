### final

关键字`final`（最终的） 标记的类`不能被继承`， 提高`安全性`，提高程序的`可读性`。

#### final修饰类

这个类就不能被继承； 如：`String`类、`StringBuffer`类、`System`类等

![finalclass](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/finalclass.pic.jpg)

#### final 修饰方法

不能被重写； 如：Object类的getClass（）

![fianlmethod](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/finalmetho.jpg)

#### final修饰属性

变为常量 属性(没有默认初始化的值)；习惯上，常量用大写字符来写！final常量一旦确定后，就禁止再次复制！

![finalproperty](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/finalproperty.pic.jpg)

#### final试用场景

##### 权限控制

给一段代码加上 final 就意味着编译器向你作出保证，这段代码`不会再被修改`；同时，这也意味着你认为这段代码已经完备并且`没有再被进行继承或重写的必要`，因此这往往会是一个需要深思熟虑的决定。


##### 类或者方法的功能确实已经完备了

对于很多的辅助性质的工具类或者方法，可能我们会考虑加上 final。这样的类有一个比较大的特点，是很可能只包含类方法而没有实例方法。

##### 子类继承和修改是一件危险的事情

在子类继承或重写某些方法后可能做一些破坏性的事情，导致子类或者父类部分也无法正常工作的情况。在这类情况下，将编号方法标记为 final 以确保稳定，可能是一种更好的做法。

##### 为了父类中某些代码一定会被执行

有时候父类中有一些关键代码是在被继承重写后必须执行的 (比如状态配置，认证等等)，否则将导致运行时候的错误。而在一般的方法中，如果子类重写了父类方法，是没有办法强制子类方法一定去调用相同的父类方法的。

```swift
class Parent {

    final func method() {
        print("开始配置")
        // ..必要的代码

        methodImpl()

        // ..必要的代码
        print("结束配置")
    }

    func methodImpl() {
        fatalError("子类必须实现这个方法")
        // 或者也可以给出默认实现
    }

}

class Child: Parent {
    override func methodImpl() {
        //..子类的业务逻辑
    }
}
```

##### 性能考虑

使用 final 的另一个重要理由是可能带来的性能改善。因为编译器能够从 final 中获取额外的信息，因此可以对类或者方法调用进行额外的优化处理。但是这个优势在实际表现中可能带来的好处其实就算与 Objective-C 的动态派发相比也十分有限，因此在项目还有其他方面可以优化 (一般来说会是算法或者图形相关的内容导致性能瓶颈) 的情况下，并不建议使用将类或者方法转为 final 的方式来追求性能的提升。

#### 参考文献

[Increasing Performance by Reducing Dynamic Dispatch](https://developer.apple.com/swift/blog/?id=27)

[FINAL](https://swifter.tips/final/)

