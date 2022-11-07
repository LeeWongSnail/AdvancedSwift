## Swift 函数式编程

### 编程范式

编程范式主要有两种

- 命令式
- 声明式

下面我们来看下这两者的定义：

```
命令式编程通过一系列改变程序状态的指令来完成计算。命令式编程模拟电脑运算，是行动导向的，关键在于定义解法，即“怎么做”，因而算法是显性而目标是隐性的。

声明式编程只描述程序应该完成的任务。声明式编程模拟人脑思维，是目标驱动的，关键在于描述问题，即“做什么”，因而目标是显性而算法是隐性的。
```
通过下面两端代码可以很明显的看出，这两种范式的区别

```swift
func commandMethod() {
    let numbers = [1,2,3,4,5,6]
    var total = 0
    for number in numbers {
        total = total + number
    }
    print(total)
}
```
这个方法很好的体现出来`怎么做`，即要想获取和就需要将数组中的每一个元素取出，然后进行累加操作

我们再来看下声明式

```swift
func decalerMethod() {
    let numbers = [1,2,3,4,5,6]
    let total = numbers.reduce(0) { $0 + $1}
    print(total)
}
```

这个方法主要体现在`做什么`,我的目标是要求和，那么我就要将数组中的元素累加即可，不需要考虑是遍历以及何种方式遍历的问题

两套代码执行的结果都是一样的，都是获取数组中元素的和，但是重点确不同，一个关注算法即怎么做，另一个关注目标 做什么。

### 函数式编程的优点

- 减轻程序员思考的负担，降低出错的可能性

每一个步骤独立，在完成目标的过程中，将整体目标拆分出一个个的小目标，每个部分所需要思考的都是独立的

- 代码可读性高
- 更简洁、更少的代码
上述两个优点，从上面的例子中可以很明显的看到，尤其是可读性，让读代码的人重点关注目标，而不是每一步如何实现

- 适用于并发环境
- 易于优化

函数式编程中每一步的实现要求是纯函数，即函数的结果只依赖参数，所以适用并发环境，而且由于每一部分分开独立，所以可进行单独的优化，而不会影响其他方法

- 细粒度的重用（函数级别）
- 易于测试

纯方很容易进行单元测试，因为只需要关心输入输出，重用也是，不用考虑调用顺序不用担心影响后续逻辑。

- 易于重构

![](https://iweiyun.github.io/images/swift-functional/1542615515_42.png)


### 柯里化

柯里化即将一个接受多参数的函数变换为一系列只接受单个参数的函数.

下面我们举一个简单的例子，

```swift
func normalFunction(v1: Int, v2: Int) -> Int {
    return v1 + v2
}

print(normalFunction(v1: 2, v2: 3))
// 5

```
这是一个我们常用的方法形式，两个参数一个返回值，

我们尝试将这个方法进行柯里化

```swift
func curryingFunction(v: Int) -> (Int) -> Int {
    return {$0 + v}
}

print(curryingFunction(v: 2)(3))
// 5
```

这里我们将多个参数变成了一个参数，不过方法的返回值从一个简单的Int值变成了一个block

如果是三个参数呢，结果应该是一个类似这种：

```swift

/// 柯里化最终版本: 传一个两个参数的函数过来, 自动柯里化
func currying<A, B, C>(_ fn: @escaping (A, B) -> C) -> (B) ->(A) -> C{
  return { b in
    return { a in
      return fn(a, b)
    }
  }
}

curring(normalFunction)(20)(10)//使用效果

```
这个方法即将normalFunction进行柯里化的方法，我们进一步简化

```swift
prefix func ~<A, B, C>(_ fn: @escaping(A, B)-> C) ->(B) ->(A) -> C {
  {b in { a in normalFunction(a, b) } }
}
print((~normalFunction)(20)(10))
```

### 一等公民和高阶函数

在函数式编程中，函数是一等公民，不再把函数想象成一个处理过程，而是把它当作一个对象或者变量来对待。

当函数是变量后，我们就可以把它作为参数或者返回值放到另一个函数中。

`所谓的高阶函数，就是把函数作为函数参数的函数`


### 高阶函数

#### Map

我们先看下map方法的实现

```swift
@inlinable
public func map<T>(
  _ transform: (Element) throws -> T
) rethrows -> [T] {
  let initialCapacity = underestimatedCount
  var result = ContiguousArray<T>()
  result.reserveCapacity(initialCapacity)

  var iterator = self.makeIterator()

  // Add elements up to the initial capacity without checking for regrowth.
  for _ in 0..<initialCapacity {
    result.append(try transform(iterator.next()!))
  }
  // Add remaining elements, if any.
  while let element = iterator.next() {
    result.append(try transform(element))
  }
  return Array(result)
}
```

从上面的实现中我们可以看到一下几点：

- 改方法的返回值是一个数组，数组中元素的类型可以发生改变也可以不发生改变 取决于transform中的内容
- 方法返回的数组中元素个数应该与原数组格式一致 append 不接受nil
- 方法的实现类似于遍历数组中每个元素并对每个元素执行相同操作，将操作的返回值作为新数组的元素

示例:

```swift
let names:[String] = ["Lee","Wong","Cendy","Luo"]
let lowCasedName = names.map { $0.lowercased()}
let characterCounts = names.map { $0.count }
print(lowCasedName)
print(characterCounts)
```

#### flatMap

还有一个方法和map相近，我们同样先看下实现

```swift
@inlinable
public func flatMap<SegmentOfResult: Sequence>(
  _ transform: (Element) throws -> SegmentOfResult
) rethrows -> [SegmentOfResult.Element] {
  var result: [SegmentOfResult.Element] = []
  for element in self {
    result.append(contentsOf: try transform(element))
  }
  return result
}
```
方法的实现基本和map是一致的，唯一的区别

```swift
result.append(contentsOf: try transform(element))
```
和
```swift
result.append(try transform(element))
```

最明显的一个区别就是第一个方法实现是将一个数组的内容添加到另一个数组中，对于一维数组来说没有区别，但是对于二维数组来说就比较明显了
下面我们尝试验证下我们的想法:

```swift
let numbers = [[1,2,3],[4,5,6],[7,8,9]]
let mapResult = numbers.map { elem in
    return elem
}
print(mapResult)
// [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

let flatMapResult = numbers.flatMap { elem in
    return elem
}
print(flatMapResult)
// [1, 2, 3, 4, 5, 6, 7, 8, 9]
```
结果很明显 我们的猜想是正确的，二维数组被拆分成了一维数组`append(contentsOf`可能是一个空数组，而append参数是一个nil

我们来用代码验证下:

```swift
let number = [1,2,3,4,5,6,7]
let flatMapResult = number.flatMap { elem -> Int? in
    if elem % 2 == 0 {
        return nil
    }
    return elem
}
print(flatMapResult)
let result = number.map { elem -> Int? in
    if elem % 2 == 0 {
        return nil
    }
    return elem
}
print(result)
```
输出结果为:

```swift
[1, 3, 5, 7]
[Optional(1), nil, Optional(3), nil, Optional(5), nil, Optional(7)]
```

由此可见，flatMap实际上还有过滤的作用，对于map的扩展相当于 返回的数组个数可以与原数组个数不同

#### compactMap

```swift
@inlinable // protocol-only
public func compactMap<ElementOfResult>(
  _ transform: (Element) throws -> ElementOfResult?
) rethrows -> [ElementOfResult] {
  return try _compactMap(transform)
}

// The implementation of compactMap accepting a closure with an optional result.
// Factored out into a separate function in order to be used in multiple
// overloads.
@inlinable // protocol-only
@inline(__always)
public func _compactMap<ElementOfResult>(
  _ transform: (Element) throws -> ElementOfResult?
) rethrows -> [ElementOfResult] {
  var result: [ElementOfResult] = []
  for element in self {
    if let newElement = try transform(element) {
      result.append(newElement)
    }
  }
  return result
}
```
这个实现就更明显了,与map的最大区别就是下面这句

```
if let newElement = try transform(element) {
  result.append(newElement)
}
```
很明显这里进行了解包操作，也就是说他可以过滤掉不符合条件(返回nil)的元素

```swift
let array = [1,2,3,"4","5"] as [Any]
let compactMapResult = array.compactMap { elem -> Any? in
    guard let str = elem as? String else { return nil }
    return str
}

print(compactMapResult)
// ["4", "5"]
```


#### Filter

我们还是先来看下[源码](https://github.com/apple/swift/blob/main/stdlib/public/core/Sequence.swift)的实现

```swift
/// - Parameter isIncluded: A closure that takes an element of the
///   sequence as its argument and returns a Boolean value indicating
///   whether the element should be included in the returned array.
/// - Returns: An array of the elements that `isIncluded` allowed.
///
/// - Complexity: O(*n*), where *n* is the length of the sequence.
@inlinable
public __consuming func filter(
  _ isIncluded: (Element) throws -> Bool
) rethrows -> [Element] {
  return try _filter(isIncluded)
}

@_transparent
public func _filter(
  _ isIncluded: (Element) throws -> Bool
) rethrows -> [Element] {
  // 新建一个ContiguousArray
  var result = ContiguousArray<Element>()
  // 创建一个遍历器
  var iterator = self.makeIterator()
  // 这里为什么用遍历器呢？？？
  while let element = iterator.next() {
    // 判断当前这个element是否满足条件
    if try isIncluded(element) {
	  // 满足条件后才添加到目标数组中
      result.append(element)
    }
  }

  return Array(result)
}
```
从这个方法中我们也可以看到:

- 方法的返回值是一个数组，数组中的元素和原数组中的元素相同
- 只有满足指定条件的元素才会被返回  这实际上是一个针对数组的筛选

总的来说Filter方法的关键就是: `过滤`

```swift
 let filterName = names.filter { elem in
     return elem.count == 3
 }
 print(filterName)
```

这里我们可以做一个扩展，因为Filter方法是过滤，那么我们就想到排序算法中的快速排序，我们可以尝试使用filter来快速写一个快排算法实现:

```swift
extension Array where Element: Comparable {
    func quickSorted() -> [Element] {
        if self.count > 1 {
            // 找到基准
			let (pivot, remaining) = (self[0], dropFirst())
			// 通过filter对数组进行筛选
            let lhs = remaining.filter{ $0 <= pivot}
            let rhs = remaining.filter{ $0 > pivot}
			// 递归
            return lhs.quickSorted() + [pivot] + rhs.quickSorted()
        }
        return self
    }
}
```

### Reduce

同样 先看[源码](https://github.com/apple/swift/blob/main/stdlib/public/core/SequenceAlgorithms.swift)

```swift
/// If the sequence has no elements, `nextPartialResult` is never executed
/// and `initialResult` is the result of the call to `reduce(_:_:)`.
///
/// - Parameters:
///   - initialResult: The value to use as the initial accumulating value.
///     `initialResult` is passed to `nextPartialResult` the first time the
///     closure is executed.
///   - nextPartialResult: A closure that combines an accumulating value and
///     an element of the sequence into a new accumulating value, to be used
///     in the next call of the `nextPartialResult` closure or returned to
///     the caller.
/// - Returns: The final accumulated value. If the sequence has no elements,
///   the result is `initialResult`.
///
/// - Complexity: O(*n*), where *n* is the length of the sequence.
@inlinable
public func reduce<Result>(
  _ initialResult: Result,
  _ nextPartialResult:
    (_ partialResult: Result, Element) throws -> Result
) rethrows -> Result {
  // 暂存原始数据
  var accumulator = initialResult
  // 对当前容器中的每个元素
  for element in self {
    // 执行传入的block，block的第一个参数就是原始数据，第二个参数为序列中的每个元素
    accumulator = try nextPartialResult(accumulator, element)
  }
  return accumulator
}
```
从上述的源码中我们可以看到

- 函数的返回值不在是数组，而是nextPartialResult的返回结果
- 这个函数实际是一个不断累计的过程，第一次执行的结果accumulator作为第二次执行的参数nextPartialResult(accumulator, element)
- accumulator用于记录每一步的记过

这让我想起累加的算法，例如

```swift
let numbers = [1,2,3,4,5,6,7,8,9,10]
let result = numbers.reduce(0) { result, elem in
    return result + elem
}
print(result)
// 55
// 1: result = 基础值 + number[0]
// 2: result = result + number[1]
```
这就是一个累加 前两个值的累加结果作为第二次运算的基准，其中0表示基础值是0 如果0改为3则输出结果为58。



### 参考文章

[一文读懂Swift函数式编程](https://zhuanlan.zhihu.com/p/192483039)
[Swift函数式编程探索](https://iweiyun.github.io/2018/11/19/swift-functional/)
[Swift-函数式编程与面向协议编程](https://juejin.cn/post/6984314556851945509#heading-1)







































