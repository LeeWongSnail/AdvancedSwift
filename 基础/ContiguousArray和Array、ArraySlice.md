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

