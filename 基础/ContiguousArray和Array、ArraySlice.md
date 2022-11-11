## ContiguousArray和Array、ArraySlice

### ContiguousArray

```
The ContiguousArray type is a specialized array that always stores its elements in a contiguous region of memory. 
This contrasts with Array, which can store its elements in either a contiguous region of memory or an NSArray 
instance if its Element type is a class or @objc protocol.

If your array’s Element type is a class or @objc protocol and you do not need to bridge the array to NSArray or 
pass the array to Objective-C APIs, using ContiguousArray may be more efficient and have more predictable 
performance than Array. If the array’s Element type is a struct or enumeration, Array and ContiguousArray should 
have similar efficiency.
```
`ContiguousArray`是一种特殊的`Array`,可以在一段`连续的`内存空间内存储数组中的元素。

当我们的数组元素是`Class`或`@objc protocol`类型的话，并且我们不需要在`Objective-C`中使用该数组的话，那么我们最好使用 `ContiguousArray`。这是因为`Array`需要额外的资源来处理跟`NSArray`的桥接功能，但是`ContiguousArray`则不需要，所以 `ContiguousArray`比`Array`的效率要高。

另外需要注意的是官方文档说如果数组元素不是`Class`和`@objc protocol`类型的话，`Array` 和 `ContiguousArray` 的效率是一样的。（我猜测是因为如果 Array 的元素都是 Struct 类型的话，它就不需要消耗资源来处理桥接的问题了。）

```
Efficiency is equivalent to that of Array, unless Element is a class or @objc protocol type, in which case using
 ContiguousArray may be more efficient.
```

不过经过测试，即使对于struct类型ContiguousArray也要比Array好一些

```swift
 let array2 = Array<Int>(1...1000000)
 let array3 = ContiguousArray<Int>(1...1000000)

 var start = CFAbsoluteTimeGetCurrent()
 array2.reduce(0, +)
 var end = CFAbsoluteTimeGetCurrent() - start
 print("Array Took \(end) seconds")

 start = CFAbsoluteTimeGetCurrent()
 array3.reduce(0, +)
 end = CFAbsoluteTimeGetCurrent() - start
 print("ContiguousArray Took \(end) seconds")
```
输出结果:

```
Array Took 0.17538797855377197 seconds
ContiguousArray Took 0.10337996482849121 seconds
```
但其实结果相差不大


### ArraySlice

```
The ArraySlice type makes it fast and efficient for you to perform operations on sections of a larger array. Instead of copying over the elements of a slice to new storage, an ArraySlice instance presents a view onto the storage of a larger array. And because ArraySlice presents the same interface as Array, you can generally perform the same operations on a slice as you could on the original array.

For more information about using arrays, see Array and ContiguousArray, with which ArraySlice shares most properties and methods.

Long-term storage of ArraySlice instances is discouraged. A slice holds a reference to the entire storage of a larger array, not just to the portion it presents, even after the original array’s lifetime ends. Long-term storage of a slice may therefore prolong the lifetime of elements that are no longer otherwise accessible, which can appear to be memory and object leakage.

```

顾名思义，ArraySlice就是Array的一个切片，ArraySlice的作用主要体现在当我们需要对一个较大数组的某一部分进行操作可以更加高效。ArraySlice是直接使用原数组的某一部分，而非将数组的某一部分拷贝出来，由于ArraySlice对外提供的接口与Array一直，因此所有对Array有效的操作对ArraySlice也同样有效。

```swift
let absences = [0, 2, 0, 4, 0, 3, 1, 0]

let midpoint = absences.count / 2

let firstHalf = absences[..<midpoint]
let secondHalf = absences[midpoint...]

let firstHalfSum = firstHalf.reduce(0, +)
let secondHalfSum = secondHalf.reduce(0, +)

if firstHalfSum > secondHalfSum {
    print("More absences in the first half.")
} else {
    print("More absences in the second half.")
}

```

![arrayslice]()

同时 

```swift
typealias SubSequence = ArraySlice<Element>
```

但是，虽然ArraySlice是某个数组的一部分，但是修改ArraySlice是并不会影响原始数组

```swift
let absences = [0, 2, 0, 4, 0, 3, 1, 0]

let midpoint = absences.count / 2

var firstHalf = absences[..<midpoint]
withUnsafePointer(to: firstHalf) { print("外部变量地址: \($0)") }
firstHalf.append(99)
withUnsafePointer(to: firstHalf) { print("外部变量地址: \($0)") }

print(absences)
//外部变量地址: 0x00007ff7b7831928
//外部变量地址: 0x00007ff7b78318f8
//[0, 2, 0, 4, 0, 3, 1, 0]
```
根据输出我们也可以很明显的看到，absences数组元素并没有因为firstHalf添加了其他元素而发生变化，且firstHalf在append操作执行前和执行后内存地址是变化的。这很可能是因为 Copy on Write。


#### 下标

对于普通的数组，我们都知道数组的下标是从0开始的，那么对于数组的切片呢？

我们通过下面的代码来验证下

```swift
let absences = [0, 2, 0, 4, 0, 3, 1, 0]

let midpoint = absences.count / 2

let firstHalf = absences[..<midpoint]
let secondHalf = absences[midpoint...]

print(firstHalf.firstIndex(where: {$0>=0}))
print(secondHalf.firstIndex(where: {$0>=0}))
```

输出结果为:

```swift
Optional(0)
Optional(4)
```

很明显，这里secondHalf的小标是继承了原始数组的下标，如果这里我们执行下面的操作:

```swift
let absences = [0, 2, 0, 4, 0, 3, 1, 0]

let midpoint = absences.count / 2

let firstHalf = absences[..<midpoint]
let secondHalf = absences[midpoint...]

print(secondHalf[0])
```
则 运行时会直接crash提示

![arraysliceindexcrash]()

结合上面我们说的，我们在来看下这种操作后呢

```swift
let absences = [0, 2, 0, 4, 0, 3, 1, 0]

let midpoint = absences.count / 2

let firstHalf = absences[..<midpoint]
var secondHalf = absences[midpoint...]

secondHalf.append(9)

print(firstHalf.firstIndex(where: {$0>=0}))
print(secondHalf.firstIndex(where: {$0>=0}))
print(secondHalf.lastIndex(where: {$0>=0}))
```

输出结果:

```swift
Optional(0)
Optional(4)
Optional(8)
```

实际上下标依然是从4开始从lastIndex我们也看出，数据添加是完成的，因此对于切片数组下标是从其初始化方法时就固定了，后续都是在初始化之后的基础上进行添加或者删除

#### 内存问题

但是我们不推荐长期持有某个大数组的切片，一个切片会增加整个大数组的引用计数，而非仅仅是指定切片部分，甚至当原始数组被释放。因此，长期持有切片可能会导致数组的其他元素的生命周期被延长，并导致内存泄漏。

```swift
var currentAbsences:ArraySlice<Int> = []


override func viewDidLoad() {
    super.viewDidLoad()
    let absences = [0, 2, 0, 4, 0, 3, 1, 0] as [Int]

    let midpoint = absences.count / 2

    let firstHalf = absences[..<midpoint]
    currentAbsences = absences[midpoint...]

}

override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)
    print(currentAbsences)
}

输出结果:
[0, 3, 1, 0]
```

我们看到absences实际上是在viewDidLoad中声明的一个局部变量，其生命周期应该就是在viewDidLoad函数执行结束后就被释放，但是我们依然可以在ViewDidAppear中打印出数组元素，这就表示absences并没有被释放，我们以为引用了数组的一部分而导致整个数组无法被释放，这就会导致内存泄露。

#### 参考文献

[Understanding The ArraySlice in Swift](https://medium.com/appcoda-tutorials/understanding-the-arrayslice-3b4957b9d965)

[ArraySlice](https://developer.apple.com/documentation/swift/arrayslice/)

[ContiguousArray](https://developer.apple.com/documentation/swift/contiguousarray/)

