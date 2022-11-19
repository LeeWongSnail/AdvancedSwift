## SIL角度看Lazy

### Lazy 属性

lazy修饰的属性我们一般称之为懒加载，即类实例构造的时候，延迟存储属性并不进行构造或者初始化，只有当开发者调用类实例的这个属性时，才完成构造或者初始化操作，主要用于降低实例初始化时间。

- lazy 属性的初始值在第一次使用的时候才进行计算
- 用关键字lazy来标识懒加载属性

### Lazy 注意点

- 懒加载的属性必须有初始值

![lazyinit]()

- 懒加载属性必须使用var 不能使用let

![lazy let]()

一个正确的懒加载属性定义应该如下:

```swift
class Person {
    lazy var name: String = "LeeWong"
}
```

### Lazy属性计算时机

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


然后执行到尾部

### lazy 属性分析

我们通过下面这段代码来生成 SIL文件,不需要编译通过

```swift
class Animal {
    lazy var age: Int = 18
}

let a = Animal()
let age = a.age

```

我们先来看下Animal类的结构体

```swift
class Animal {
  lazy var age: Int { get set }
  @_hasStorage @_hasInitialValue final var $__lazy_storage_$_age: Int? { get set }
  @objc deinit
  init()
}
```

从上面的定义可以看出 age是一个试用final修饰的可选型属性.

接下来我们在来看下age的初始化

```swift
// variable initialization expression of Animal.$__lazy_storage_$_age
sil hidden [transparent] @$s6setSil6AnimalC21$__lazy_storage_$_age33_4819FA22CA2D1D51637BDBC63D4493F9LLSiSgvpfi : $@convention(thin) () -> Optional<Int> {
bb0:
  %0 = enum $Optional<Int>, #Optional.none!enumelt // user: %1
  return %0 : $Optional<Int>                      // id: %1
} // end sil function '$s6setSil6AnimalC21$__lazy_storage_$_age33_4819FA22CA2D1D51637BDBC63D4493F9LLSiSgvpfi'
```

从上面代码中我们可以看到age的初始值是`optional.none`,类似OC中的nil。

我们在来看下age的get方法

```swift
// Animal.age.getter
sil hidden [lazy_getter] [noinline] @$s6setSil6AnimalC3ageSivg : $@convention(method) (@guaranteed Animal) -> Int {
// %0 "self"                                      // users: %14, %2, %1
bb0(%0 : $Animal):
  debug_value %0 : $Animal, let, name "self", argno 1, implicit // id: %1
  %2 = ref_element_addr %0 : $Animal, #Animal.$__lazy_storage_$_age // user: %3
  %3 = begin_access [read] [dynamic] %2 : $*Optional<Int> // users: %4, %5
  %4 = load %3 : $*Optional<Int>                  // user: %6
  end_access %3 : $*Optional<Int>                 // id: %5
  switch_enum %4 : $Optional<Int>, case #Optional.some!enumelt: bb1, case #Optional.none!enumelt: 
bb2 // id: %6

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
} // end sil function '$s6setSil6AnimalC3ageSivg'

```


### 参考文献

[Swift(十一)-Swift中的lazy](https://juejin.cn/post/7061033820954296334)

