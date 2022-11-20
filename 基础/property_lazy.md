## 属性-Lazy


lazy修饰的属性我们一般称之为懒加载，即类实例构造的时候，延迟存储属性并不进行构造或者初始化，只有当开发者调用类实例的这个属性时，才完成构造或者初始化操作，主要用于降低实例初始化时间。

- 使用lazy修饰的存储属性

- 延迟属性必须有一个默认的初始值

- 延迟存储在第一次访问的时候才被赋值

- 延迟存储属性并不能保证线程安全

- 延迟存储属性对实例对象大小的影响


#### 使用lazy修饰的存储属性

一个正确的懒加载属性定义应该如下:

```swift
class Person {
    lazy var name: String = "LeeWong"
}
```

#### 懒加载的属性必须有初始值

![lazyinit](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/lazynoinit.png)

#### 懒加载属性必须使用var 不能使用let

![lazy let](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/lazylet.png)



#### 延迟存储在第一次访问的时候才被赋值

我们通过执行下面这段代码，并根据代码执行结果来分析

```swift
class Person {
    lazy var name: String = "LeeWong"
}
private func test() {
    let h = Human()
    h.age = 2
    print(h.age)
}
}
```

首先执行到h.age=2
![step1](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/lazystep1.png)

然后执行到尾部

![step2](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/lazystep2.png)

从上面的输出我们可以确定lazy修饰的属性，对象初始化后并不会被赋值，而是等到第一次被使用时才会被赋值。


##### 通过SIL来看lazy的实现

首先我们使用

```
swiftc -emit-sil main.swift | xcrun swift-demangle >> ./main.sil && code main.sil
```
这个命令对我们的源码进行还原，还原后的完整文件在[这里]()

下面我们来看看这个文件中的内容:

###### lazy 实际是optional

```
class Animal {
  lazy var age: Int { get set }
  @_hasStorage @_hasInitialValue final var $__lazy_storage_$_age: Int? { get set }
  @objc deinit
  init()
}
```
通过查看SIL之后的代码，我们可以发现age实际上被定义为Int?,即age是一个可选型 其默认值为Optional.none即nil

###### lazy 的第一次调用

我们先来看下get方法的实现

```

// Animal.age.getter
sil hidden [lazy_getter] [noinline] @setSil.Animal.age.getter : Swift.Int : $@convention(method) (@guaranteed Animal) -> Int {
// %0 "self"                                      // users: %14, %2, %1
bb0(%0 : $Animal):
  debug_value %0 : $Animal, let, name "self", argno 1, implicit // id: %1
  %2 = ref_element_addr %0 : $Animal, #Animal.$__lazy_storage_$_age // user: %3
  %3 = begin_access [read] [dynamic] %2 : $*Optional<Int> // users: %4, %5
  %4 = load %3 : $*Optional<Int>                  // user: %6
  end_access %3 : $*Optional<Int>                 // id: %5
  switch_enum %4 : $Optional<Int>, case #Optional.some!enumelt: bb1, case #Optional.none!enumelt: bb2 // id: %6

// %7                                             // users: %9, %8
bb1(%7 : $Int):                                   // Preds: bb0
  debug_value %7 : $Int, let, name "tmp1", implicit // id: %8
  br bb3(%7 : $Int)                               // id: %9

bb2:                                              // Preds: bb0
  %10 = integer_literal $Builtin.Int64, 18        // user: %11
  %11 = struct $Int (%10 : $Builtin.Int64)        // users: %18, %13, %12
  debug_value %11 : $Int, let, name "tmp2", implicit // id: %12
  %13 = enum $Optional<Int>, #Optional.some!enumelt, %11 : $Int // user: %16
  %14 = ref_element_addr %0 : $Animal, #Animal.$__lazy_storage_$_age // user: %15
  %15 = begin_access [modify] [dynamic] %14 : $*Optional<Int> // users: %16, %17
  store %13 to %15 : $*Optional<Int>              // id: %16
  end_access %15 : $*Optional<Int>                // id: %17
  br bb3(%11 : $Int)                              // id: %18

// %19                                            // user: %20
bb3(%19 : $Int):                                  // Preds: bb2 bb1
  return %19 : $Int                               // id: %20
} // end sil function 'setSil.Animal.age.getter : Swift.Int'

```

首先默认进入的是bb0，首先我们已知道age的初始值是optional.none,我们只需要重点关注:

```
  switch_enum %4 : $Optional<Int>, case #Optional.some!enumelt: bb1, case #Optional.none!enumelt: bb2 // id: %6
```

即

```
switch $Optional<Int> {
case Optional.some:
    bb1
case Optional.none:
    bb2
}
```
由于我们的默认值是none，所以我们继续关注bb2：

```
bb2:                                              // Preds: bb0
  %10 = integer_literal $Builtin.Int64, 18        // user: %11
  %11 = struct $Int (%10 : $Builtin.Int64)        // users: %18, %13, %12
  debug_value %11 : $Int, let, name "tmp2", implicit // id: %12
  %13 = enum $Optional<Int>, #Optional.some!enumelt, %11 : $Int // user: %16
  %14 = ref_element_addr %0 : $Animal, #Animal.$__lazy_storage_$_age // user: %15 这里
  %15 = begin_access [modify] [dynamic] %14 : $*Optional<Int> // users: %16, %17
  store %13 to %15 : $*Optional<Int>              // id: %16
  end_access %15 : $*Optional<Int>                // id: %17
  br bb3(%11 : $Int) 
```

在这个方法里，才把1这个值赋值给了age属性。


#### 延迟存储属性并不能保证线程安全

根据上面sil文件，主要是查看age的getter方法，如果此时有两个线程：

- 线程1此时访问age，其age是没有值的，进入bb2流程

- 然后时间片将CPU分配给了线程2，对于optional来说，依然是none，同样可以走到bb2流程

- 所以，在此时，线程1会走一遍赋值，线程2也会走一遍赋值，并不能保证属性只初始化了一次

#### 延迟存储属性对实例对象大小的影响

通过上面我们知道，lazy的属性在使用时才被初始化，那么lazy修饰的属性是否会影响实例对象的内存大小呢？

我们可以通过这个方法打印实例对象内存大小:

```swift
print(class_getInstanceSize(Human.self))
```

首先我们不添加lazy修饰

```swift
class Human {
    var age: Int = 1
}
private func test() {
    let h = Human()
    print(class_getInstanceSize(Human.self))
	//24
}

```

我们添加lazy后

```swift
class Human {
    lazy var age: Int = 1
}
private func test() {
    let h = Human()
    print(class_getInstanceSize(Human.self))
	//32
}
```
从上面的输出我们可以看到使用lazy修饰的确是会影响类的实例的内存大小占用，因此lazy是可以节省内存的。



### lazy能否重写get和set

如果一个属性我们实现了get方法和可选的set方法，那么这个属性就是一个计算属性，
因此 对于lazy修饰的属性，我们没办法重写set或者get方法

`注意`: 对于一个属性来说，一旦自定义了get方法，那么这个属性就从存储属性变为了计算属性。对于计算属性来说，你不可以只有set方法

![propertyonlyset](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/propertyonlyset.jpeg)



### 参考文献

[Swift(十一)-Swift中的lazy](https://juejin.cn/post/7061033820954296334)

[完整的sil后的代码](https://github.com/LeeWongSnail/AdvancedSwift/blob/main/res/lazysil.txt)

