## Range

### 区间

`范围`和`区间`是基本相同的概念构建的（一个连续元素的系列，有开始有结尾），但使用了不同的方法。`范围基于索引`，因此可以是个`集合`，他们的大多数功能都是基本这个特性的。`区间不是集合`，他们的实现是依赖Comparable协议的。我们只可以为服从Comparable协议的类型创建区间类型

很遗憾当我们尝试查找`HalfOpenInterval`和`ClosedInterval `时发现最新版本的Swift已经没有这两个类型了(Swift 3.1移除)。

所以，Swift 3.1以后无论是Interval还是Range都统称为Range。


我们先来看下区间运算符的几种用法

#### ClosedRange ...

可以简单的表示为 [a, b]

```swift
@_transparent
public static func ... (minimum: Self, maximum: Self) -> ClosedRange<Self> {
  _precondition(
    minimum <= maximum, "Range requires lowerBound <= upperBound")
  return ClosedRange(_uncheckedBounds: (lower: minimum, upper: maximum))
}
```
表示包含startIndex和endIndex的区间

```swift
for iCount in 512...1024{
     //从512遍历到1024（包括1024）
}
```

#### Range ..<

可简单的表示为 [a, b)


```swift
/// Returns a half-open range that contains its lower bound but not its upper
/// bound.
///
/// Use the half-open range operator (`..<`) to create a range of any type
/// that conforms to the `Comparable` protocol. This example creates a
/// `Range<Double>` from zero up to, but not including, 5.0.
///
///     let lessThanFive = 0.0..<5.0
///     print(lessThanFive.contains(3.14))  // Prints "true"
///     print(lessThanFive.contains(5.0))   // Prints "false"
///
/// - Parameters:
///   - minimum: The lower bound for the range.
///   - maximum: The upper bound for the range.
///
/// - Precondition: `minimum <= maximum`.
public static func ..< (minimum: ClosedRange<Bound>.Index, maximum: ClosedRange<Bound>.Index) -> Range<ClosedRange<Bound>.Index>
```

表示包含startIndex但是不包含endIndex的区间,通用用在遍历数组下标时

```swift
var fruts = ["apple","orange","banana"]
let iCount = fruts.count
for i in 0..<iCount{
    print("第\(i+1)个水果是\(fruts[i])")
}
```

### 单侧区间

可简单表示为 (-infinite, b] 或者 【a, +infinite)

闭区间操作符有另一个表达形式，可以表达往一侧无限延伸的区间。

例如，一个包含了数组从索引 2 到结尾的所有值的区间[2...]，或者从开头到index为2为止[...2]

```swift
let names = ["lee","wong","cendy","luo"]
for name in names[...2] {
    print(name)
}
/// lee
/// wong
/// cendy

for name in names[2...] {
    print(name)
}
/// cendy
/// luo
```

单侧区间是闭区间操作符的一种，所以起始和结束的都包含。


#### 倒序循环

通过reversed()可以将一个正序循环变成逆序循环

```
for i in (0..<10).reversed() {
    print(i)
}
```

#### 字符串的区间

```swift
//字符串截取
let words = "leewong.cn"

//字符串截取
let index = words.index(words.startIndex, offsetBy: 4)
let index2 = words.index(words.startIndex, offsetBy: 6)
let range1 = Range(uncheckedBounds: (lower: index, upper: index2))
let newRangeStr1 = words[..<index]
let newRangeStr2 = words[index...]
print(newRangeStr1)
// leew
print(newRangeStr2)
// ong.cn
```

### 其他运算


#### 是否包含
因为Range表示一个区间，因此也可以判断某个数是否在某个区间内

```swift
let test = "a"
let interval = "a"..."z"
if interval.contains(test) {
    print(test)
}

// a
```

#### 获取字符串子串

```
extension String {
    subscript(r: ClosedRange<Int>) -> String {
        let start = index(startIndex, offsetBy: r.lowerBound)
        let end = index(startIndex, offsetBy: r.upperBound)
        return self[start...end]
    }
    
    subscript(r: Range<Int>) -> String {
        get {
            let start = index(startIndex, offsetBy: r.lowerBound)
            let end = index(startIndex, offsetBy: r.upperBound)
            return self[start..<end]
        }
    }
}
```

使用上面的方法我们可以很容易的获取字符串的子串

#### 字符串截取

如果我们没办法通过下标去获取字符串的子串，那么我们就需要进行转换

```swift
let str = "LeeWong"
let index = str.firstIndex(of: "W") ?? str.startIndex
let subString = str[index...]
print(subString)
/// Wong
```

### 参考文献
[Range -Swift（译）](https://www.jianshu.com/p/b804f0090a74)
[Swift 中的Range类型和 Range运算符](https://blog.csdn.net/BoilerUp/article/details/109018875)
[Swift 3のRange徹底解説](https://qiita.com/mono0926/items/88779ceff30f8fc705c5)


