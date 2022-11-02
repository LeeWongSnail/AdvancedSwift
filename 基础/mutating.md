## mutating And inout

在平时的开发过程中，我们经常会写到这样的代码

```
class Student {
    var name: String?
    var point: Point?
    func changeName(newName: String) {
        name = newName
    }
}
```
`class`提供给外部一个方法用来修改自身的某个属性。由于在Swift中引入了`struct`，因此很多时候我们的一些模型都是用`struct`。那么如果我们上面这个方法使用`struct`实现呢？

```swift
struct SeniorStudent {
    var name: String?
    var point: Point?
    func changeName(newName: String) {
        name = newName
    }
}
```
很遗憾，编译器会提示我们因为`struct`本身是值类型，所以我们无法在方法中修改自己。即`self is immutable`

那么如果我们想要修改这个属性应该如何操作呢？这时候就用到了`mutating`关键词

mutating关键词可以加载值类型中方法的前面，表示可以在这个方法中修改值类型本身。

如上代码，在进行mutating添加后编译器就不在提示错误

```swift
mutating func changeName(newName: String) {
    name = newName
}
```

那么如果对于引用类型呢？编译器提示`mutating is not valid on instance methods in class`,因为对于引用类型(class)来说,本身就是可以在内部修改自身属性的，所以不需要增加mutating关键词。

![mutatingininstancemethod](https://github.com/LeeWongSnail/AdvancedSwift/blob/main/res/941667403487_.pic.jpg?raw=true)


因此mutating，经常用在某些即可能在类中又可能在结构体中实现的方法，一般是协议中定义的方法。如下

```swift
protocol ChangeNameProtocol {
    mutating func changeName(newName: String)
}
```
这样的话 无论是类还是结构体遵守这个协议时都可以实现这个方法：

```swift
protocol ChangeNameProtocol {
    mutating func changeName(newName: String)
}


class Student {
    var name: String?
    var point: Point?
    func changeName(newName: String) {
        name = newName
    }
}

struct SeniorStudent {
    var name: String?
    var point: Point?
    mutating func changeName(newName: String) {
        name = newName
    }
}
```
`注意`:
- 类中实现protocol中对应mutating修饰的方法不需要写mutating关键词，因为在引用类型中本来就支持在方法中修改自身
- 结构体中实现protocol中对应mutating方法必须写mutating关键词，否则提示报错


### mutating的实现

通过上面的描述我们基本了解了mutating的作用和使用场景及意义，那么mutating是如何实现的呢？为什么添加了这个关键词我们就可以在值类型的方法中修改自身了呢？

首先我们必须明确的一点是在`引用类型的方法中为什么我们可以修改自身的值？` 其实很简单引用类型我们只复制指针，但实际指向的内存区域是同一个，所以我们可以直接修改，但是值类型是值的复制，修改复制后的变量不会影响之前的变量，因此我们无法修改。

下面我们使用sil对源码进行解析，看下是mutating是如何实现的

为了对比我们先看下没有加mutating修饰的代码sil之后的

```swift
// 这时候代码是报错的
struct CollageStudent {
    var name: String?
    func changeName(newName: String) {
        name = newName
    }
}
```
sil后

```
// CollageStudent.changeName(newName:)
sil hidden @$s7SilTest14CollageStudentV10changeName03newF0ySS_tF : $@convention(method) (@guaranteed String, @guaranteed CollageStudent) -> () {
// %0 "newName"                                   // users: %12, %10, %2
// %1 "self"                                      // user: %3
bb0(%0 : $String, %1 : $CollageStudent):
  debug_value %0 : $String, let, name "newName", argno 1 // id: %2
  debug_value %1 : $CollageStudent, let, name "self", argno 2, implicit // id: %3
  %4 = integer_literal $Builtin.Word, 1           // user: %6
  // function_ref _allocateUninitializedArray<A>(_:)
  %5 = function_ref @$ss27_allocateUninitializedArrayySayxG_BptBwlF : $@convention(thin) <τ_0_0> (Builtin.Word) -> (@owned Array<τ_0_0>, Builtin.RawPointer) // user: %6
  %6 = apply %5<Any>(%4) : $@convention(thin) <τ_0_0> (Builtin.Word) -> (@owned Array<τ_0_0>, Builtin.RawPointer) // users: %8, %7
  %7 = tuple_extract %6 : $(Array<Any>, Builtin.RawPointer), 0 // user: %14
  %8 = tuple_extract %6 : $(Array<Any>, Builtin.RawPointer), 1 // user: %9
  %9 = pointer_to_address %8 : $Builtin.RawPointer to [strict] $*Any // user: %11
  retain_value %0 : $String                       // id: %10
  %11 = init_existential_addr %9 : $*Any, $String // user: %12
  store %0 to %11 : $*String                      // id: %12
  // function_ref _finalizeUninitializedArray<A>(_:)
  %13 = function_ref @$ss27_finalizeUninitializedArrayySayxGABnlF : $@convention(thin) <τ_0_0> (@owned Array<τ_0_0>) -> @owned Array<τ_0_0> // user: %14
  %14 = apply %13<Any>(%7) : $@convention(thin) <τ_0_0> (@owned Array<τ_0_0>) -> @owned Array<τ_0_0> // users: %23, %20
  // function_ref default argument 1 of print(_:separator:terminator:)
  %15 = function_ref @$ss5print_9separator10terminatoryypd_S2StFfA0_ : $@convention(thin) () -> @owned String // user: %16
  %16 = apply %15() : $@convention(thin) () -> @owned String // users: %22, %20
  // function_ref default argument 2 of print(_:separator:terminator:)
  %17 = function_ref @$ss5print_9separator10terminatoryypd_S2StFfA1_ : $@convention(thin) () -> @owned String // user: %18
  %18 = apply %17() : $@convention(thin) () -> @owned String // users: %21, %20
  // function_ref print(_:separator:terminator:)
  %19 = function_ref @$ss5print_9separator10terminatoryypd_S2StF : $@convention(thin) (@guaranteed Array<Any>, @guaranteed String, @guaranteed String) -> () // user: %20
  %20 = apply %19(%14, %16, %18) : $@convention(thin) (@guaranteed Array<Any>, @guaranteed String, @guaranteed String) -> ()
  release_value %18 : $String                     // id: %21
  release_value %16 : $String                     // id: %22
  release_value %14 : $Array<Any>                 // id: %23
  %24 = tuple ()                                  // user: %25
  return %24 : $()                                // id: %25
} // end sil function '$s7SilTest14CollageStudentV10changeName03newF0ySS_tF'
```
我们先只关注参数部分

```swift

  debug_value %0 : $String, let, name "newName", argno 1 // id: %2
  debug_value %1 : $CollageStudent, let, name "self", argno 2, implicit // id: %3
```

虽然我们只是显示声明了一个参数但是解析出来仍有一个隐藏的参数(感觉类似OC中消息发送，消息接受者是一个默认参数)。参数newName是一个String类型的不可变参数，还有一个CollageStudent类型的不可变参数self。

然后我们在看下使用mutating修饰之后的方法

```swift
struct CollageStudent {
    var name: String?
    mutating func changeName(newName: String) {
        print(newName)
    }
}
```
sil之后

```
sil hidden [transparent] @$s7SilTest14CollageStudentV4nameSSSgvs : $@convention(method) (@owned Optional<String>, @inout CollageStudent) -> () {

// %0 "value"                                     // users: %11, %8, %4, %2
// %1 "self"                                      // users: %5, %3
bb0(%0 : $Optional<String>, %1 : $*CollageStudent):
  debug_value %0 : $Optional<String>, let, name "value", argno 1, implicit // id: %2
  debug_value %1 : $*CollageStudent, var, name "self", argno 2, implicit, expr op_deref // id: %3
  retain_value %0 : $Optional<String>             // id: %4
  %5 = begin_access [modify] [static] %1 : $*CollageStudent // users: %10, %6
  %6 = struct_element_addr %5 : $*CollageStudent, #CollageStudent.name // users: %8, %7
  %7 = load %6 : $*Optional<String>               // user: %9
  store %0 to %6 : $*Optional<String>             // id: %8
  release_value %7 : $Optional<String>            // id: %9
  end_access %5 : $*CollageStudent                // id: %10
  release_value %0 : $Optional<String>            // id: %11
  %12 = tuple ()                                  // user: %13
  return %12 : $()                                // id: %13
} // end sil function '$s7SilTest14CollageStudentV4nameSSSgvs'
```

首先第一个感觉应该是，比未添加之前代码量少了好多，下面我们在对比下参数

```
  debug_value %0 : $Optional<String>, let, name "value", argno 1, implicit // id: %2
  debug_value %1 : $*CollageStudent, var, name "self", argno 2, implicit, expr op_deref // id: %3
```
我们发现函数的第二个参数self变成了var,即变量，也就是说使用了mutating修饰后我们的self变成了一个可变的。

而且我还发现方法名也有变化

未使用mutating修饰的方法名:

```
sil hidden @$s7SilTest14CollageStudentV10changeName03newF0ySS_tF : $@convention(method) (@guaranteed String, @guaranteed CollageStudent) -> () {
```
使用mutating修饰后的方法名:

```
sil hidden [transparent] @$s7SilTest14CollageStudentV4nameSSSgvs : $@convention(method) (@owned Optional<String>, @inout CollageStudent) -> () {
```
通过对比我们发现未使用mutating修饰的方法名前面是`@guaranteed`修饰,而使用muatting修饰后的方法名使用的是 `@owned`和`@inout`修饰。

我们把重点放在inout这个关键词上，这个关键词的作用是什么呢？可能这就是muatting的关键。

### inout

`inout是Swift中的关键字，可以放在参数类型前，冒号之后。使用inout之后，函数体内部可以直接修改参数值，且会保留改变。`

我们都知道，无论是值类型还是引用类型在作为函数参数使用时，默认都是不可变的，一旦我们想要修改参数的值，Xcode就会提示我们`cannot assign to value: 'param' is a 'let' constant`. 

而inout就是为了让我们避免上面的问题而生。

在研究inout之前，我们先分辨针对值类型和引用类型在作为方法参数时的传递做一个验证

#### 值类型

```swift
func makeStudentNameDefault(stu: SeniorStudent) -> SeniorStudent {
    withUnsafePointer(to: stu) { print("地址2: \($0)") }
    var innerStu = stu
    innerStu.name = "Empty"
    return innerStu
}

func testInout() {
    var stu = SeniorStudent()
    stu.name = "Lee"
    print(stu)
    withUnsafePointer(to: &stu) { print("地址1: \($0)") }
    print(makeStudentNameDefault(stu: stu))
    print(stu)
    withUnsafePointer(to: &stu) { print("地址3: \($0)") }
}
```

打印结果：

```
SeniorStudent(name: Optional("Lee"), point: nil)
地址1: 0x00007ff7b5b48ac0
地址2: 0x00007ff7b5b48920
SeniorStudent(name: Optional("Empty"), point: nil)
SeniorStudent(name: Optional("Lee"), point: nil)
地址3: 0x00007ff7b5b48ac0
```
在`makeStudentNameDefault`中操作的`innerStu`由于是一个结构体即值类型，所以是值传递`innerStu`是一个新的实例。修改了innerStud并不会影响外部的值。

#### 引用类型

```swift
func makeStudentNameDefault(stu: Student) -> Student {
    withUnsafePointer(to: stu) { print("参数地址: \($0)") }
    print("地址1: \(Unmanaged.passUnretained(stu).toOpaque())")
    stu.name = "Empty"
    return stu
}

func testInout() {
    var stu = Student()
    stu.name = "Lee"
    withUnsafePointer(to: stu) { print("外部变量地址: \($0)") }
    print(stu.name)
    print("地址2: \(Unmanaged.passUnretained(stu).toOpaque())")
    print(makeStudentNameDefault(stu: stu).name)
    print(stu.name)
    print("地址3: \(Unmanaged.passUnretained(stu).toOpaque())")
}
```

打印结果

```
外部变量地址: 0x00007ff7b0f0ab58
Optional("Lee")
地址2: 0x000060000236c440
参数地址: 0x00007ff7b0f0a928
地址1: 0x000060000236c440
Optional("Empty")
Optional("Empty")
地址3: 0x000060000236c440
```
首先我们对比先变量地址：testInout函数中stu的指针地址和makeStudentNameDefault函数中参数的stud地址不同，这说明作为参数传递时，引用类型新建了一个指针

然后我们在对比对象的地址，发现无论是刚创建对象时，还是在makeStudentNameDefault函数内部又或是修改后的testInout函数内，地址都是一样的。

也就是说对于引用类型来说，作为参数会新建一个变量，但是依旧是指向之前对象的内存地址，所以函数内修改依然会影响函数外部。

看完了引用类型和值类型，然后我们在来看下重点`inout`

首先还是值类型

```
func makeStudentNameDefault(stu: inout SeniorStudent) -> SeniorStudent {
    withUnsafePointer(to: stu) { print("地址2: \($0)") }
    stu.name = "Empty"
    return stu
}

func testInout() {
    var stu = SeniorStudent()
    stu.name = "Lee"
    print(stu)
    withUnsafePointer(to: &stu) { print("地址1: \($0)") }
    print(makeStudentNameDefault(stu: &stu))
    print(stu)
    withUnsafePointer(to: &stu) { print("地址3: \($0)") }
}
```
相较于之前的代码只是修改了两个位置:

- 参数添加inout修饰
- 参数传递添加取地址符&

我们先来看下输出结果:

```
SeniorStudent(name: Optional("Lee"), point: nil)
地址1: 0x00007ff7bdc4faf0
地址2: 0x00007ff7bdc4f958
SeniorStudent(name: Optional("Empty"), point: nil)
SeniorStudent(name: Optional("Empty"), point: nil)
地址3: 0x00007ff7bdc4faf0
```
通过地址我们发现，添加了inout修饰后，即使是值类型，现在也是引用传递了，也就是说在函数内部可以修改外部参数了(通打印的name被修改也可以验证)

对于引用参数这里我不在列举其效果与未添加时是一致的，因此inout关键词对于引用类型的参数意义不大。

通过上面的验证我们了解到

- 使用inout关键字的函数，在调用时需要在该参数前加上&符号
- 使用inout的参数在传入时必须为变量，不能为常量 
`Cannot pass immutable value as inout argument: 'stu' is a 'let' constant`

- 使用inout的参数不能有默认值，不能为可变参数
`Cannot provide default value to inout parameter 'stu'`
- 使用inout的参数不等同与函数返回值，是一种使参数的作用域超出函数体的方式
- 多个使用inout的参数不能同时传入一个变量，因为拷入拷出的顺序不定，那么最终值也不能确定

### inout的实现

* 如果实参有物理内存地址，且没有设置属性观察器，直接将实参的内存地址传入函数（实参进行引用传递）

* 如果实参是计算属性或设置了属性观察器，采取Copy In Copy Out的做法：
- 调用该函数时，先复制实参的值，产生一个副本（局部变量-执行get方法）
- 将副本的内存地址传入函数（副本进行引用传递），在函数内部可以修改副本的值
- 函数返回后，再将副本的值覆盖实参的值（执行set方法）

copy in copy out简单来说就是：

```
var c = a;
change(c)
a = c
```

### 参考文献

[‘mutating’ in Swift](https://agrawalsuneet.medium.com/mutating-in-swift-7327d8a1cddd)
[inout的本质](https://juejin.cn/post/6946377209695666190)
[mutating & inout](https://juejin.cn/post/7103692507229192205#heading-1)

