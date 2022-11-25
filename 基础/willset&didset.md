## 属性观察者

本文主要讨论继承的属性一些相关方法的调用逻辑，主要涉及:

- willset
- didset

下面是本文中用到的类

```swift
class Father {
    var fatherName: String?
    var name: String? {
        willSet {
            print("Father Name willSet")
        }
        
        didSet {
            print("Father Name didSet")
        }
    }
    var age: Int?
}

class Son: Father {
    override var name: String? {
        willSet {
            print("Son Name willSet")
        }
        
        didSet {
            print("Son Name didSet")
        }
    }
}
```

每当有观察者的属性被修改时就会调用willSet和didSet

### 继承情况下属性观察者调用顺序

```swift
private func inheritTest() {
    let son = Son()
    son.name = "Wong"
}
// Son Name willSet
// Father Name willSet
// Father Name didSet
// Son Name didSet
```

结论: 调用顺序为子类`子类 willSet`->`父类 will Set` ->`父类 didSet` -> `子类 didSet`

### 初始化是否导致观察者方法被调用

```swift
class Father {
    var fatherName: String?
    init(nickName: String) {
        name = nickName
    }
    var name: String {
        willSet {
            print("Father Name willSet")
        }

        didSet {
            print("Father Name didSet")
        }
    }
    var age: Int?
}

private func inheritTest() {
    let father = Father(nickName: "LeeWong")
}
//  
```
为什么在init方法中不会调用didSet或者willSet方法呢？

我们在官方的文档中找到了答案:

```swift
NOTE

The willSet and didSet observers of superclass properties are called
 when a property is set in a subclass initializer, after the 
superclass initializer has been called. They aren’t called while a
 class is setting its own properties, before the superclass 
initializer has been called.


For more information about initializer delegation, see Initializer
 Delegation for Value Types and Initializer Delegation for Class
 Types.
```
根据对上面描述的理解，willSet和didSet在类superClass被调用之前，设置本身属性的时候不会被调用。

但是上文同时也提到了子类给父类属性设置值的时候会触发，我们来看下:

```swift
class Father {
    var name: String? {
        willSet {
            print("Father Name willSet")
        }

        didSet {
            print("Father Name didSet")
        }
    }
}

class Son: Father {
    override init() {
        super.init()
        name = "Lee"
    }
}

let son = Son()

//Father Name willSet
//Father Name didSet

```
输出结果也验证了，文档中的描述，子类初始化方法中修改了父类的属性，是会触发父类属性的观察器的，因为此时父类已经初始化完成。

### 类和结构体的属性观察者

```swift
class HealthInfo { // 请注意，这是一个 class
    var height: Float {
        didSet {
            print(height)
        }
    }
    var weight: Float
    init(height: Float, weight: Float) {
        self.height = height
        self.weight = weight
    }
}
class User {
    var healthInfo: HealthInfo {  // 请注意观察这个属性的 didSet
        didSet {
            print(healthInfo)
        }
    }
    init(healthInfo: HealthInfo) {
        self.healthInfo = healthInfo
    }
}
let user = User(healthInfo: .init(height: 160, weight: 50))
user.healthInfo.height = 162
user.healthInfo.height = 163

// 162.0
// 163.0
```

这里有两个类HealthInfo和User，我们分别观察了其中的height属性和healthInfo属性

在来看下另外一个例子，我们只是把HealthInfo改为了结构体

```swift
struct HealthInfo { // 请注意，这是一个 struct
    var height: Float {
        didSet {
            print(height)
        }
    }
    var weight: Float
    init(height: Float, weight: Float) {
        self.height = height
        self.weight = weight
    }
}
class User {
    var healthInfo: HealthInfo {  // 请注意观察这个属性的 didSet
        didSet {
            print(healthInfo)
        }
    }
    init(healthInfo: HealthInfo) {
        self.healthInfo = healthInfo
    }
}
let user = User(healthInfo: .init(height: 160, weight: 50))
user.healthInfo.height = 162
user.healthInfo.height = 163

// 162.0
// HealthInfo(height: 162.0, weight: 50.0)
// 163.0
// HealthInfo(height: 163.0, weight: 50.0)
```
为什么只是将类改为结构体，会发生如此区别呢？

首先我们先明确下: 结构体是值类型， 类是引用类型。修改结构体就是修改对应内存地址的值，而修改类是修改类中属性指针指向的存储空间的值。

所以，HealthInfo为类时修改HealthInfo中的height并不会导致healthInfo被修改，但是HealthInfo未结构体时，修改height会实实在在的修改到healthInfo。因此就会调用到User中的healthInfo的属性观察者。

### 总结

didset&willset,一般在属性赋值后都会被调用
但是在下面几个场景中不会被调用:

- 在初始化方法中初始化当前类的属性不会调用，但是在子类中修改父类属性会调用
- 如果被观察的属性是引用类型，则修改引用类型中的某个字段不会调用引用类型的观察方法，当然如果在mutating方法中修改了self也是会被调用


### 参考文献

[swift探索2: swift属性](https://blog.csdn.net/changcongcong_ios/article/details/120001439)

[Swift didSet 为什么没有执行？](https://blog.ficowshen.com/page/post/5)

