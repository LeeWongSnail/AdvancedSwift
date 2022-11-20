## 底层窥探let和var修饰变量的区别

这边文章我们通过SIL来看下let和var属性底层实现的区别

对于下面这个类:

```swift
class Animal {
    let id: String = "123"
    var name: String?
}
```

经过SIL命令后转变为:

```
class Animal {
  @_hasStorage @_hasInitialValue final let id: String { get }
  @_hasStorage @_hasInitialValue var name: String? { get set }
  @objc deinit
  init()
}
```

通过上面的对比我们发现:

- let 修饰的属性自动添加了 final关键词修饰
- let 和 var 都是用了@_hasStorage @_hasInitialValue修饰
- let 修饰的属性只有get方法，而var修饰的属性有get和set方法

下面我们来一一分析这几个区别:

### final

我们都知道final关键词， final表明let修饰的存储属性是不可以继承的。

当我们尝试在子类重写父类的let属性是我们发现:

```swift
class Child {
    let age = 1
    var name: String?
}

class Baby: Child {
    override let age = 1
}
```

上述代码在编译时提示

![overridelet]()

`Cannot override with a stored property 'age'`,虽然是我们预料之中的报错，但是这个提示确并非我们希望的。这是什么原因呢？

我们在来尝试下重写var修饰的属性呢？

```swift
override var name: String?
```
果然

![overridevar]()

依然提示`Cannot override with a stored property 'name'`,难道子类不能重载父类的属性？？？

网上查了很多资料貌似在Swift 4.1之后就不允许这么写了， 就提原因不太清楚。

这里有几个参考的文档

[Cannot override with a stored property in Swift](https://stackoverflow.com/questions/49640147/cannot-override-with-a-stored-property-in-swift)
[For 'lazy', make "cannot override with a stored property" a warning #13304](https://github.com/apple/swift/pull/13304/commits/31e0dfbcc7191b0675c00ada7e38bfc2416531f8)

但是如果想要重写父类的属性，我们可以这么写:

```swift
class Baby: Child {
    override var name: String? {
        get {
            return ""
        }
        set {
            
        }
    }
}
```

甚至我们可以修改只读属性为可读写属性
例如:

```swift
class Child {
    let age = 1
    var name: String? {
        return "super"
    }
}

class Baby: Child {
    private var tmp: String?
    override var name: String? {
        get {
            return tmp
        }
        set {
            tmp = newValue
        }
    }
}
```

这样我们父类中的name就变成了可读写的属性。但是实际上我们使用了一个private的属性tmp进行了存储。所以这实际意义上并不能真正达到修改父类属性的读写性。


### @_hasStorage @_hasInitialValue

这个和属性包装器的使用类似

@_hasStorage：表示这个属性是存储属性
@_hasInitialValue: 表示已初始化

### let 只有get方法

首先因为let修饰的属性只有get方法，因此let修饰的属性是不可以被更改的

### let 真的不可修改？

我们先来看下下面这段代码:

```swift
func randomName(isBoy: Bool) -> String {
    let childName: String
    if isBoy {
        childName = "LeeWong"
    } else {
        childName = "Cendy"
    }
    return childName
}

func test() {
    let name = randomName(isBoy: true)
    print("\(name)")
    //  LeeWong
}
```

猜测下这段代码是否有什么问题呢？

实际上这段代码可以正确运行并输出对应的结果！ WHAT？？ childName不是let吗？为什么是可变？？？

虽然是赋值操作，但是if和else指回进入一个，赋值具有唯一性，不可能同时赋值两个，例如我们在ObjectMapper源码中看到的

```swift
/// Convert a JSON String into an Object using NSJSONSerialization
public static func parseJSONString(JSONString: String) -> Any? {
    let data = JSONString.data(using: String.Encoding.utf8, allowLossyConversion: true)
    if let data = data {
        let parsedJSON: Any?
        do {
            parsedJSON = try JSONSerialization.jsonObject(with: data, options: JSONSerialization.ReadingOptions.allowFragments)
        } catch let error {
            print(error)
            parsedJSON = nil
        }
        return parsedJSON
    }
    return nil
}
```

这时候我们再来品味下let的定义: `let 用来声明常量，常量的值一旦设置好便不能再被更改`


### 参考文献

[Swift：let与var](https://juejin.cn/post/7019113636740235295)
[Swift 5.1 (13) - 继承](https://juejin.cn/post/6844904068616290318)
[swift底层探索 02 - 属性swift底层探索 02 - 属性](https://cloud.tencent.com/developer/article/1858054)
[Swift 进阶：属性](https://juejin.cn/post/7048595981201309732)


