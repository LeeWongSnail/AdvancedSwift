## 可选型 Optional

### 定义

```
A type that represents either a wrapped value or nil, the absence of a value.

```
官网对于Optional的描述是这样的，其实简单一句话来概括就是这个类型可以有值也可以为空

问题： 入什么要引入Optional呢？
在OC中，一个对象如果为nil，是没有什么影响的，因为OC的方法调用底层是msg_send，对一个空对象发送消息是不会有任何问题的。但Swift是一个强类型的语言(类型安全)，如果一个对象被申明为String类型那么他就不可以为nil。 及时String和String?两者也不是同一个类型。

对于nil的理解OC和Swift也不同
- OC中的nil: 表示缺少一个合法的对象，一个指针指向了一个空对象，
- Swift中nil: 表示任意类型的值缺失，是一个确定值，当声明一个可选变量或者属性时没提供默认值即表示nil(Optional.none)
- Swift中的可选型类似OC中的nil,但是OC中nil只对class有用，但是可选型在Swift中对一类型都可用，也更安全

例如:

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7j2qbnzscj319s05sdgf.jpg)

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7j2spwgytj31by066gml.jpg)

我们通过查看Optional的声明也可以验证上面的猜想

```swift
@frozen public enum Optional<Wrapped> : ExpressibleByNilLiteral {

    /// The absence of a value.
    ///
    /// In code, the absence of a value is typically written using the `nil`
    /// literal rather than the explicit `.none` enumeration case.
    case none

    /// The presence of a value, stored as `Wrapped`.
    case some(Wrapped)
}
```



### 如何声明一个可选型

我们最长用的声明一个可选型的方法即:

```swift
var str2: String?
```
当然我们还可以用另外一种方式来声明

```swift
var str3: Optional<String>
```

即： `? = Optional<T>` 

如果我们在声明的时候没有设置默认值则实际上定义相当于

```swift
var str4: String? = Optional<String>.none
==
var str3: Optional<String> = nil
```

### 解包

#### 强解包

可以通过在可选型值后面添加！来对一个可选型进行强制解包

```swift
var str2: String?
print(str2!)
```
但是，一旦被强解包的变量说着属性为nil,则强解包会导致运行时崩溃。因此一般不建议直接强解包


#### 自动解包

使用前进行为空判断

```swift
if str3 != nil {
    print(str3)
}
```

#### 可选绑定(推荐)

这是目前最推荐的一种使用方式，我们通常会通过下面两种方式进行判断:

- if let：如果有值，则会进入if 流程
- gurad let：如果为nil，则会进入else流程
即

```swift
if let str = str3 {
    print(str)
}
guard let str = str3 else { return }
print(str)
```

#### 隐式解包

由于我们使用可选型后，每次使用变量或者属性时都要进行为空判断，这样会让我们的逻辑有点复杂，而且是大量重复判断，很低效。因此我们在定义变量或者属性时，对于某些我们确定是有值的可以这么定义:

```swift
var name: String!
```
这样我们在使用的时候就不需要显示的判断解包, 也可以与其他非可选类型进行直接操作

```swift
var name: String!
var name2 = "abc"
let res = name + name2
```

但是这里要注意，即使你写明了是肯定存在的，上面代码中定义的name也是可选型，这就意味着我依然可以将这个值设置为nil

```swift
var name: String! = nil
```

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7jkh9wjt5j30rq08e0t3.jpg)

因此一旦这个可选型为nil, 就还是有可能会导致程序crash

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7jkm0oys5j31kq04yt9d.jpg)

#### ?? 运算符

这个运算符实际与我们在OC中使用的三母运算符功能类似，在OC中，当某个值可能为空时，可以为其设置一个默认值

```swift 
NSInteger res = a != nil ? a : b

```

使用?? 后我们可以简化成

```swift
let res = a ?? b
```
即 如果a为nil这使用b

我们来看下?? 这个方法是如何实现的

```swift
@_transparent
public func ?? <T>(optional: T?, defaultValue: @autoclosure () throws -> T)
    rethrows -> T {
  switch optional {
  case .some(let value):
    return value
  case .none:
    return try defaultValue()
  }
}
```


#### unsafelyUnwrapped

```swift
/// The wrapped value of this instance, unwrapped without checking whether
  /// the instance is `nil`.
  ///
  /// The `unsafelyUnwrapped` property provides the same value as the forced
  /// unwrap operator (postfix `!`). However, in optimized builds (`-O`), no
  /// check is performed to ensure that the current instance actually has a
  /// value. Accessing this property in the case of a `nil` value is a serious
  /// programming error and could lead to undefined behavior or a runtime
  /// error.
  ///
  /// In debug builds (`-Onone`), the `unsafelyUnwrapped` property has the same
  /// behavior as using the postfix `!` operator and triggers a runtime error
  /// if the instance is `nil`.
  ///
  /// The `unsafelyUnwrapped` property is recommended over calling the
  /// `unsafeBitCast(_:)` function because the property is more restrictive
  /// and because accessing the property still performs checking in debug
  /// builds.
  ///
  /// - Warning: This property trades safety for performance.  Use
  ///   `unsafelyUnwrapped` only when you are confident that this instance
  ///   will never be equal to `nil` and only after you've tried using the
  ///   postfix `!` operator.
  @inlinable
  public var unsafelyUnwrapped: Wrapped {
    @inline(__always)
    get {
      if let x = self {
        return x
      }
      _debugPreconditionFailure("unsafelyUnwrapped of nil optional")
    }
  }
```

这个也很简单，看方法试下就明白，这个属性当可选型非nil时都是正常的，但是当可选值是nil，时就会提示`unsafelyUnwrapped of nil optional`错误，这实际上跟强制解包是一样的。

不过从注释的描述看:`However, in optimized builds (-O), no check is performed to ensure that the current instance actually has a value. Accessing this property in the case of a nil value is a serious programming error and could lead to undefined behavior or a runtime error.
`

我们先直接运行下面这段代码：

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7jue8x9p5j326y03ywf0.jpg)

我们尝试对工程做以下修改:

1、修改Build Setting里的 op level 

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7jucfgl5ej321003oq3u.jpg)

2、修改运行时的模式为release

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7jucezs3fj31ca0fwjt7.jpg)

然后我们在来运行代码
![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7juce85ouj327q08kac2.jpg)

好像的确是不报错了,不过Console里提示可能有风险。而且我们仔细看注释的时候会发现`his property trades safety for performance`,这个属性实际上是用`安全换性能`，使用可能会导致出现安全问题。

### 可选型遵守的协议

#### Equatable

这也说明了，可选型是可以比较是否相等的

```swift
extension Optional: Equatable where Wrapped: Equatable {
  @_transparent
  public static func ==(lhs: Wrapped?, rhs: Wrapped?) -> Bool {
    switch (lhs, rhs) {
    case let (l?, r?):
      return l == r
    case (nil, nil):
      return true
    default:
      return false
    }
  }
}
```

可选型比较的逻辑也比较简单，如果两个都非nil，则直接比较大小，如果两个都为nil则返回true，如果其中一个为nil则返回false


### 参考文献

[Swift 可选类型Optional](https://www.jianshu.com/p/47e2fd84aaa6)

[wift之深入解析可选类型Optional的底层原理](https://blog.csdn.net/Z1591090/article/details/120613913)


