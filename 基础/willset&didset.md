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
通过上面方法我们得出，在init中给有监听者的属性赋值并不会调用到观察者相关方法



### 参考文献

[swift探索2: swift属性](https://blog.csdn.net/changcongcong_ios/article/details/120001439)