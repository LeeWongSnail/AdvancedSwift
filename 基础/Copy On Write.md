## Copy On Write

### 背景

在Swift中，字典(Dictionary)、数组(Array)以及结构体都不在是引用类型，而变为值类型。但是作为容器类，我们可能会在数组和字典中存放大量的数据，

值类型的一个特点就是在传递和赋值时会进行复制操作，每次赋值肯定会产生额外的开销，尤其值类型还是存放在栈中的，这很有可能导致极大的栈空间浪费。

### 示例

```swift
func address(o: UnsafeRawPointer) -> String {
    return String.init(format: "%018p", Int(bitPattern: o))
}

func valueTypeTest() {
    var a = [1,2,3]
    print(address(o: &a))
    var b = a
    print(address(o: &b))
}
```

打印结果为:

```
0x0000600000565aa0
0x0000600000565aa0
```

a/b两个数组的值竟然是相同的，也就是三个值指向的同一块内存空间，`这好像与值类型每次赋值时都会复制相违背`，下面我们尝试修改数组内容

```swift
func valueTypeTest() {
    var a = [1,2,3]
    print(address(o: &a))
    var b = a
    print(address(o: &b))
    
    b.append(4)
    print("address of b \(address(o: &b))")
}
```
打印结果为

```
0x0000600003e0f560
0x0000600003e0f560
address of b 0x000060000085d880
```

### 分析

从上面的打印结果我们发现，在给b添加一个元素后，b的内存地址发生了变化，也就是说，实际在赋值的时候并未真正的发生值复制，而是在修改的时候才发生。

这就是Swift中的COW(Copy On Write)即写时复制。这样在一些不经常发生变化的场景下，COW可以很大程度上提高我们的内存操作，有效减少了内存的分配和回收，也不会浪费栈上宝贵的内存资源。

### 总结

在使用数组和字典时的最佳实践应该是，按照具体的数据规模和操作特点来决定到时是使用值类型的容器还是引用类型的容器：

- 在需要处理大量数据并且频繁操作 (增减) 其中元素时，选择 NSMutableArray 和 NSMutableDictionary 会更好，
- 对于容器内条目小而容器本身数目多的情况，应该使用 Swift 语言内建的 Array 和 Dictionary。