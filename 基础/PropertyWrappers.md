## Property Wrappers

```
A property wrapper adds a layer of separation between code that manages how a property is stored and the code that defines a property.
```

属性包装器是Swift语言中的新特性，他是我们能够自定义类型，并在各处使用，该类型默认实现get和set功能。属性包装器用来修饰属性，他可以抽取关于属性重复的逻辑来达到简化代码的目的。


### 属性包装器的应用

首先我们通过一个例子来看下属性包装器的使用场景以及如何使用属性包装器来简化代码。

例如我们App中有很多引导，我们需要记录展示状态在偏好中，我们会经常写下面的代码:

```swift
extension UserDefaults {

    public enum Keys {
        static let hadShownGuideView = "had_shown_guide_view"
    }

    var hadShownGuideView: Bool {
        set {
            set(newValue, forKey: Keys.hadShownGuideView)
        }
        get {
            return bool(forKey: Keys.hadShownGuideView)
        }
    }
}

/// 下面的就是业务代码了。
let hadShownGuide =  UserDefaults.standard.hadShownGuideView 
if !hadShownGuide {
    /// 显示新手引导 并保存本地为已显示
    showGuideView() /// showGuideView具体实现略。
    UserDefaults.standard.hadShownGuideView = true
}
```

但是我们App中有很多引导，那么我们每个地方都需要如此实现，就造成了很大成都的代码冗余。那么我们怎么利用属性包装来简化代码呢？

首先我们，我们在有多个重复代码时，现将上面相同的东西抽象出来

```swift
@propertyWrapper /// 先告诉编译器 下面这个UserDefault是一个属性包裹器
struct UserDefault<T> {
    ///这里的属性key 和 defaultValue 还有init方法都是实际业务中的业务代码   
    ///我们不需要过多关注
    let key: String
    let defaultValue: T

    init(_ key: String, defaultValue: T) {
        self.key = key
        self.defaultValue = defaultValue
    }
///  wrappedValue是@propertyWrapper必须要实现的属性
/// 当操作我们要包裹的属性时  其具体set get方法实际上走的都是wrappedValue 的set get 方法。 
    var wrappedValue: T {
        get {
            return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
        }
        set {
            UserDefaults.standard.set(newValue, forKey: key)
        }
    }
}

///封装一个UserDefault配置文件
struct UserDefaultsConfig {
///告诉编译器 我要包裹的是hadShownGuideView这个值。
///实际写法就是在UserDefault包裹器的初始化方法前加了个@
/// hadShownGuideView 属性的一些key和默认值已经在 UserDefault包裹器的构造方法中实现
  @UserDefault("had_shown_guide_view", defaultValue: false)
  static var hadShownGuideView: Bool
}

///具体的业务代码。
UserDefaultsConfig.hadShownGuideView = false
print(UserDefaultsConfig.hadShownGuideView) // false
UserDefaultsConfig.hadShownGuideView = true
print(UserDefaultsConfig.hadShownGuideView) // true
```

实际上上面我们只做了三件事
- 使用@propertyWrapper修饰了一个新的类型
- 将重复代码进行抽象
- 将有重复逻辑的属性使用抽象后的类型修饰@UserDefault

### Property Wrapper使用

您可以将property wrapper视为常规属性，它将get和set方法委托给其他类型。

对于属性包装器类型有两个要求:

- 必须使用属性@propertyWrapper进行定义
- 他必须具有wrappedValue属性

下面是属性包装器最简单的样子:

```swift
@propertyWrapper
struct Wrapper<T> {
   var wrappedValue: T
}
```

现在我们就可以直接使用这个属性包装器:

```swift
struct HasWrapper {
    @Wrapper var x: Int
}

let a = HasWrapper(x: 0)
```


### Property Wrapper初始化

我们可以使用下面两种方式来初始化属性包装器

```swift
struct HasWrapperWithInitialValue {
    @Wrapper var x = 0 // 1
    @Wrapper(wrappedValue: 0) var y // 2
}
```

- 编译器隐式地调用init(wrappedValue:)用0初始化x。
- 初始化方法被明确指定为属性的一部分。


### 访问 Property Wrapper

假设我们在属性包装器中添加了一个方法，如下:

```swift
@propertyWrapper
struct Wrapper<T> {
    var wrappedValue: T

    func foo() { print("Foo") }
}
```

#### 下划线

我们可以通过在变量名前添加下划线(_)来访问包装器类型:

```swift
struct HasWrapper {
    @Wrapper var x = 0

    func foo() { _x.foo() }
}
```

`注意`: 包装器中的方法是private的，因此只有在HasWrapper内可以这样调用，

```swift
let a = HasWrapper()
a._x.foo() // ❌ '_x' is inaccessible due to 'private' protection level
```

#### projectedValue

通过定义projectedValue尚需经，属性包装器可以公开更多API，对projectedValue的类型没有任何限制。

```swift
@propertyWrapper
struct Wrapper<T> {
    var wrappedValue: T

    var projectedValue: Wrapper<T> { return self }

    func foo() { print("Foo") }
}
```

这种情况下外部可以直接使用,注意结构体实例在访问包装器属性是使用`$`

```swift
let a = HasWrapper()
a.$x.foo() // Prints 'Foo'
```

总之，我们可以使用下面三种方式访问属性包装器:

```swift
struct HasWrapper {
    @Wrapper var x = 0
    
    func foo() {
        print(x) // `wrappedValue`
        print(_x) // wrapper type itself
        print($x) // `projectedValue`
    }
}
```

### 使用限制

`Property wrappers`并非没有限制。 他们强加了许多限制：

- 带有包装器的属性不能在子类中覆盖。
- 具有包装器的属性不能是`lazy`，`@NSCopying`，`@NSManaged`，`weak`或`unowned`。
- 具有包装器的属性不能具有自定义的set或get方法。
- `wrappedValue`，`init（wrappedValue :`)和`projectedValue`必须具有与包装类型本身相同的访问控制级别
- 不能在协议或扩展中声明带有包装器的属性。


### 参考文献

[Swift 中的 propertyWrapper](https://kingcos.me/posts/2022/swift_property_wrapper/)

[Swift Property Wrappers](https://nshipster.com/propertywrapper/)

[Swift 5 属性包装器Property Wrappers完整指南](https://juejin.cn/post/6844904018121064456)


