## Swift实践

### 利用编译检查

#### 减少使用Any/AnyObject

因为Any/AnyObject缺少`明确的类型信息`，编译器无法进行`类型检查`，会带来一些问题：

- 编译器无法检查类型是否正确保证`类型安全`
- 代码中大量的`as?`转换
- 类型的缺失导致编译器无法做一些潜在的`编译优化`

##### 使用as?带来的问题
这个需要后续完善

#### 使用自定义类型代替Dictionary

代码中大量`Dictionary`数据结构会降低代码可维护性，同时带来潜在的bug：

- key需要`字符串硬编码`，编译时无法检查
- value`没有类型限制`。修改时类型无法限制，读取时需要`重复类型转换`和`解包`操作
- 无法利用`空安全`特性，指定某个属性必须有值

`自定义类型还有个好处，例如JSON转自定义类型时会进行类型/nil/属性名检查，可以避免将错误数据丢到下一层。`

不推荐

```swift
let dic: [String: Any]
let num = dic["value"] as? Int
dic["name"] = "name"
```

推荐:

```swift
struct Data {
  let num: Int
  var name: String?
}
let num = data.num
data.name = "name"

```

##### 适合使用Dictionary的场景

- 数据不使用 - 数据并不读取只是用来传递。
- 解耦 
	- 1.组件间通信解耦使用HashMap传递参数进行通信。
	- 2.跨技术栈边界的场景，混合栈间通信/前后端通信使用HashMap/JSON进行通信。

#### 使用枚举关联值代替Any

例如使用枚举改造`NSAttributedStringAPI`，原有`APIvalue`为`Any`类型无法限制特定的类型。

优化前：

```swift 
let string = NSMutableAttributedString()
string.addAttribute(.foregroundColor, value: UIColor.red, range: range)
```

优化后：

```swift
enum NSAttributedStringKey {
  case foregroundColor(UIColor)
}
let string = NSMutableAttributedString()
string.addAttribute(.foregroundColor(UIColor.red), range: range) // 不传递Color会报错
```

#### 使用泛型/协议关联类型代替Any

使用`泛型`或`协议关联类型`代替`Any`，通过泛型类型约束来使编译器进行更多的类型检查。


#### 使用枚举/常量代替硬编码

代码中存在重复的`硬编码字符串/数字`，在修改时可能会因为不同步引发bug。尽可能`减少硬编码字符串/数字`，使用`枚举`或`常量`代替。


#### 使用KeyPath代替字符串硬编码

`KeyPath`包含`属性名`和`类型信息`，可以避免`硬编码字符串`，同时当属性名或类型改变时编译器会进行检查。

不推荐

```swift
class SomeClass: NSObject {
    @objc dynamic var someProperty: Int
    init(someProperty: Int) {
        self.someProperty = someProperty
    }
}
let object = SomeClass(someProperty: 10)
object.observeValue(forKeyPath: "", of: nil, change: nil, context: nil)
```

推荐:

```swift
let object = SomeClass(someProperty: 10)
object.observe(\.someProperty) { object, change in
}
```

### 内存安全

#### 减少使用!属性

`!属性`会在读取时`隐式强解包`，当值不存在时产生运行时异常导致Crash。

```swift
class ViewController: UIViewController {
    @IBOutlet private var label: UILabel! // @IBOutlet需要使用!
}
```

#### 减少使用!进行强解包

使用!强解包会在值不存在时产生运行时异常导致Crash。
```swift
var num: Int?
let num2 = num! // 错误
```

`提示`：建议只在小范围的局部代码段使用!强解包。

#### 避免使用try!进行错误处理

使用`try!`会在方法抛出异常时产生运行时异常导致Crash。
```swift
try! method()
````

#### 使用weak/unowned避免循环引用

```swift
resource.request().onComplete { [weak self] response in
  guard let self = self else {
    return
  }
  let model = self.updateModel(response)
  self.updateUI(model)
}

resource.request().onComplete { [unowned self] response in
  let model = self.updateModel(response)
  self.updateUI(model)
}
```

#### 减少使用unowned

`unowned`在值不存在时会产生运行时异常导致Crash，只有在确定self一定会存在时才使用unowned。

```swift
class Class {
    @objc unowned var object: Object
    @objc weak var object: Object?
}
```
`unowned/weak`区别：

`weak` - 必须设置为可选值，会进行弱引用处理性能更差。`会自动设置为nil`

`unowned` - 可以不设置为可选值，不会进行弱引用处理性能更好。但是不会自动设置为nil, `如果self已释放会触发错误`.

#### 错误处理方式

- 可选值 - 调用方并不关注内部可能会发生错误，当发生错误时返回nil
- `try/catch` - 明确提示调用方需要处理异常，需要实现Error协议定义明确的错误类型
- `assert` - 断言。只能在Debug模式下生效
- `precondition` - 和assert类似，可以再`Debug/Release`模式下生效
- `fatalError` - 产生运行时崩溃会导致Crash，应避免使用
- `Result` - 通常用于闭包异步回调返回值

#### 减少使用可选值

可选值的价值在于通过明确标识值可能会为nil并且编译器强制对值进行nil判断。但是不应该随意的定义可选值，可选值不能用let定义，并且使用时必须进行解包操作相对比较繁琐。在代码设计时应考虑这个值是否有可能为nil，只在合适的场景使用可选值。

使用init注入代替可选值属性

不推荐

```
class Object {
  var num: Int?
}
let object = Object()
object.num = 1
```

推荐

```
class Object {
  let num: Int

  init(num: Int) {
    self.num = num
  }
}
let object = Object(num: 1)
```

#### 避免随意给予可选值默认值

在使用可选值时，通常我们需要在可选值为nil时进行异常处理。有时候我们会通过给予可选值默认值的方式来处理。但是这里应考虑在什么场景下可以给予默认值。在不能给予默认值的场景应当及时使用return或抛出异常，避免错误的值被传递到更多的业务流程。

不推荐

```swift
func confirmOrder(id: String) {
	// 给予错误的值会导致错误的值被传递到更多的业务流程
	confirmOrder(id: orderId ?? "")
}
```

推荐

```swift
func confirmOrder(id: String) {

	guard let orderId = orderId else {
    	// 异常处理
    	return
	}
	confirmOrder(id: orderId)
}
```

`提示`：通常强业务相关的值不能给予默认值：例如商品/订单id或是价格。`在可以使用兜底逻辑的场景使用默认值，例如默认文字/文字颜色`。

#### 使用枚举优化可选值

Object结构同时只会有一个值存在：

优化前

```swift
class Object {
    var name: Int?
    var num: Int?
}
```

优化后

- 降低内存占用 - 枚举关联类型的大小取决于最大的关联类型大小
- 逻辑更清晰 - 使用enum相比大量使用if/else逻辑更清晰

```swift
enum CustomType {
    case name(String)
    case num(Int)
}
```

#### 减少var属性

使用计算属性

使用计算属性可以减少多个变量同步带来的潜在bug。

不推荐

```swift
class model {
  var data: Object?
  var loaded: Bool
}
model.data = Object()
loaded = false
```

推荐

```swift
class model {
  var data: Object?
  var loaded: Bool {
    return data != nil
  }
}
model.data = Object()
```

`提示`：计算属性因为每次都会重复计算，所以计算过程需要`轻量`避免带来性能问题。


### 控制流

#### 使用`filter/reduce/map`代替`for`循环

使用`filter/reduce/map`可以带来很多好处，包括`更少的局部变量`，`减少模板代码`，代码更加`清晰`，`可读性更高`。

不推荐

```swift
let nums = [1, 2, 3]
var result = []
for num in nums {
    if num < 3 {
        result.append(String(num))
    }
}
// result = ["1", "2"]
```

推荐

```swift
let nums = [1, 2, 3]
let result = nums.filter { $0 < 3 }.map { String($0) }
// result = ["1", "2"]
```

#### 使用guard进行提前返回

推荐

```swift
guard !a else {
    return
}
guard !b else {
    return
}
// do
```

不推荐

```swift
if a {
    if b {
        // do
    }
}
```

#### 使用三元运算符?:

推荐

```swift
let b = true
let a = b ? 1 : 2

let c: Int?
let b = c ?? 1
```

不推荐

```swift
var a: Int?
if b {
    a = 1
} else {
    a = 2
}
```

#### 使用for where优化循环

for循环添加`where`语句，只有当where条件满足时才会进入循环

不推荐

```swift
for item in collection {
  if item.hasProperty {
    // ...
  }
}
```

推荐

```swift
for item in collection where item.hasProperty {
  // item.hasProperty == true，才会进入循环
}
```


#### 使用defer

defer可以保证在函数退出前一定会执行。可以使用defer中实现退出时一定会执行的操作例如资源释放等避免遗漏。

```swift
func method() {
    lock.lock()
    defer {
        lock.unlock()
        // 会在method作用域结束的时候调用
    }
    // do
}
```

### 字符串

使用`"""`
在定义复杂字符串时，使用多行字符串字面量可以保持原有字符串的换行符号/引号等特殊字符，不需要使用\进行转义。

```swift
let quotation = """
The White Rabbit put on his spectacles.  "Where shall I begin,
please your Majesty?" he asked.

"Begin at the beginning," the King said gravely, "and go on
till you come to the end; then stop."
"""
```

`提示`：上面字符串中的`""`和`换行`可以自动保留。

#### 使用字符串插值

使用字符串插值可以提高代码可读性。

不推荐

```swift
let multiplier = 3
let message = String(multiplier) + "times 2.5 is" + String((Double(multiplier) * 2.5))
```

推荐

```swift
let multiplier = 3
let message = "\(multiplier) times 2.5 is \(Double(multiplier) * 2.5)"
```

#### 集合

使用标准库提供的高阶函数

不推荐
```swift
var nums = []
nums.count == 0
nums[0]
```

推荐
```swift
var nums = []
nums.isEmpty
nums.first
```

### 访问控制

Swift中默认访问控制级别为`internal`。编码中应当尽可能减小属性/方法/类型的访问控制级别隐藏内部实现。

`提示`：同时也有利于编译器进行优化。

#### 使用private/fileprivate修饰私有属性和方法

```swift
private let num = 1
class MyClass {
    private var num: Int
}
```

#### 使用private(set)修饰外部只读/内部可读写属性

```swift
class MyClass {
    private(set) var num = 1
}
let num = MyClass().num
MyClass().num = 2 // 会编译报错
```


### 函数

#### 使用参数默认值
使用参数默认值，可以使调用方传递更少的参数。

不推荐

```swift
func test(a: Int, b: String?, c: Int?) {
}
test(1, nil, nil)
```

推荐

```swift
func test(a: Int, b: String? = nil, c: Int? = nil) {
}
test(1)
```

`提示`：相比ObjC，参数默认值也可以让我们定义更少的方法。

#### 限制参数数量

当方法参数过多时考虑使用自定义类型代替。

不推荐

```swift
func f(a: Int, b: Int, c: Int, d: Int, e: Int, f: Int) {
}

```

推荐

```swift
struct Params {
    let a, b, c, d, e, f: Int
}
func f(params: Params) {
}
```

#### 使用@discardableResult

某些方法使用方并不一定会处理返回值，可以考虑添加@discardableResult标识提示Xcode允许不处理返回值不进行warning提示。

```swift
// 上报方法使用方不关心是否成功
func report(id: String) -> Bool {} 

@discardableResult func report2(id: String) -> Bool {}

report("1") // 编译器会警告
report2("1") // 不处理返回值编译器不会警告
```

### 元组

避免过长的元组

元组虽然具有类型信息，但是并不包含变量名信息，使用方并不清晰知道变量的含义。所以当元组数量过多时考虑使用自定义类型代替。

```swift
func test() -> (Int, Int, Int) {

}

let (a, b, c) = test()
// a，b，c类型一致，没有命名信息不清楚每个变量的含义
```

### 系统库

`KVO/Notification` 使用 block API

block API的优势：

- KVO 可以支持 KeyPath
- 不需要主动移除监听，observer释放时自动移除监听

不推荐

```swift
class Object: NSObject {
  init() {
    super.init()
    addObserver(self, forKeyPath: "value", options: .new, context: nil)
    NotificationCenter.default.addObserver(self, selector: #selector(test), name: NSNotification.Name(rawValue: ""), object: nil)
  }

  override class func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
  }

  @objc private func test() {
  }

  deinit {
    removeObserver(self, forKeyPath: "value")
    NotificationCenter.default.removeObserver(self)
  }

}
```

推荐

```swift
class Object: NSObject {

  private var observer: AnyObserver?
  private var kvoObserver: NSKeyValueObservation?

  init() {
    super.init()
    observer = NotificationCenter.default.addObserver(forName: NSNotification.Name(rawValue: ""), object: nil, queue: nil) { (_) in 
    }
    kvoObserver = foo.observe(\.value, options: [.new]) { (foo, change) in
    }
  }
}
```

### Protocol

### 使用protocol代替继承

Swift中针对protocol提供了很多新特性，例如`默认实现，关联类型，支持值类型`。在代码设计时可以优先考虑使用protocol来避免臃肿的父类同时更多使用值类型。

`提示`：一些无法用protocol替代继承的场景：
- 1.需要继承NSObject子类。
- 2.需要调用super方法。
- 3.实现抽象类的能力。

### Extension

使用extension组织代码

使用extension将私有方法/父类方法/协议方法等不同功能代码进行分离更加清晰/易维护。

```swift
class MyViewController: UIViewController {
  // class stuff here
}
// MARK: - Private
extension: MyViewController {
    private func method() {}
}
// MARK: - UITableViewDataSource
extension MyViewController: UITableViewDataSource {
  // table view data source methods
}
// MARK: - UIScrollViewDelegate
extension MyViewController: UIScrollViewDelegate {
  // scroll view delegate methods
}
```

### 代码风格

良好的代码风格可以提高代码的可读性，统一的代码风格可以降低团队内相互理解成本。对于Swift的代码格式化建议使用自动格式化工具实现，将自动格式化添加到代码提交流程，通过定义Lint规则统一团队内代码风格。考虑使用SwiftFormat和SwiftLint。

`提示`：`SwiftFormat`主要关注代码样式的格式化，`SwiftLint`可以使用`autocorrect`自动修复部分不规范的代码。

常见的自动格式化修正:
- 移除多余的;
- 最多只保留一行换行
- 自动对齐空格
- 限制每行的宽度自动换行

### 性能优化

性能优化上主要关注提高运行时性能和降低二进制体积。需要考虑如何更好的使用Swift特性，同时提供更多信息给编译器进行优化。

#### 使用Whole Module Optimization

当Xcode开启`WMO`优化时，编译器可以将整个程序编译为`一个文件`进行更多的优化。例如通过推断`final/函数内联/泛型特化`更多使用静态派发，并且可以移除部分未使用的代码。

#### 使用源代码打包

当我们使用组件化时，为了提高编译速度和打包效率，通常单个组件独立编译生成静态库，最后多个组件直接使用静态库进行打包。这种场景下`WMO`仅针对`internal`以内作用域生效，对于`public/open`缺少外部使用信息所以无法进行优化。所以对于大量使用Swift的项目，使用全量代码打包更有利于编译器做更多优化。

#### 减少方法动态派发

- 使用final - class/方法/属性申明为final，编译器可以优化为静态派发
- 使用private - 方法/属性申明为private，编译器可以优化为静态派发
- 避免使用dynamic - dynamic会使方法通过ObjC消息转发的方式派发
- 使用WMO - 编译器可以自动分析推断出final优化为静态派发


#### 使用Slice共享内存优化性能

在使用`Array/String`时，可以使用`Slice`切片获取一部分数据。`Slice`保存对原始`Array/String`的引用共享内存数据，不需要重新分配空间进行存储。

```swift
let midpoint = absences.count / 2

let firstHalf = absences[..<midpoint]
let secondHalf = absences[midpoint...]
// firstHalf/secondHalf并不会复制和占用更多内存
```

`提示`：应避免一直持有Slice，Slice会延长原始Array/String的生命周期导致无法被释放造成内存泄漏。

#### protocol添加AnyObject

```swift
protocol AnyProtocol {}

protocol ObjectProtocol: AnyObject {}
```

当protocol仅限制为class使用时，继承AnyObject协议可以使编译器不需要考虑值类型实现，提高运行时性能。

#### 使用@inlinable进行方法内联优化

```swift
// 原始代码
let label = UILabel().then {
    $0.textAlignment = .center
    $0.textColor = UIColor.black
    $0.text = "Hello, World!"
}
```

以then库为例，他使用闭包进行对象初始化以后的相关设置。但是 then 方法以及闭包也会带来额外的性能消耗。

#####内联优化

```swift
@inlinable
public func then(_ block: (Self) throws -> Void) rethrows -> Self {
    try block(self)
    return self
}
```

```swift
// 编译器内联优化后
let label = UILabel() 
label.textAlignment = .center
label.textColor = UIColor.black
label.text = "Hello, World!"
```

#### 属性

使用lazy延时初始化属性

```swift
class View {
    var lazy label: UILabel = {
        let label = UILabel()
        self.addSubView(label)
        return label
    }()
}
```

lazy属性初始化会延迟到第一次使用时，常见的使用场景：

- 初始化比较耗时
- 可能不会被使用到
- 初始化过程需要使用self


`提示`：lazy属性不能保证线程安全

#### 避免使用private let属性

`private let`属性会增加每个class对象的`内存大小`。同时会`增加包大小`，因为需要为属性生成相关的信息。可以考虑使用文件级private let申明或static常量代替。

不推荐

```swift
class Object {
    private let title = "12345"
}
```

推荐

```swift
private let title = "12345"
class Object {
    static let title = ""
}
```

`提示`：这里并不包括通过init初始化注入的属性。

#### 使用didSet/willSet时进行Diff

某些场景需要使用didSet/willSet属性检查器监控属性变化，做一些额外的计算。但是由于didSet/willSet并不会检查新/旧值是否相同，可以考虑添加新/旧值判断，只有当值真的改变时才进行运算提高性能。

优化前

```swift
class Object {
    var orderId: String? {
        didSet {
            // 拉取接口等操作
        }
    }
}
```

例如上面的例子，当每一次orderId变更时需要重新拉取当前订单的数据，但是当orderId值一样时，拉取订单数据是无效执行。

优化后

```swift
class Object {
    var orderId: String? {
        didSet {
            // 判断新旧值是否相等
            guard oldValue != orderId else {
                return
            }
            // 拉取接口等操作
        }
    }
}
```

### 集合

集合使用lazy延迟序列

```swift
var nums = [1, 2, 3]
var result = nums.lazy.map { String($0) }
result[0] // 对1进行map操作
result[1] // 对2进行map操作
```

在集合操作时使用lazy，可以将数组运算操作推迟到第一次使用时，避免一次性全部计算。

`提示`：`例如长列表，我们需要创建每个cell对应的视图模型，一次性创建太耗费时间。`

#### 使用合适的集合方法优化性能

不推荐

```swift
var items = [1, 2, 3]
items.filter({ $0 > 1 }).first // 查找出所有大于1的元素，之后找出第一个
```

推荐

```swift
var items = [1, 2, 3]
items.first(where: { $0 > 1 }) // 查找出第一个大于1的元素直接返回
```

####  使用值类型

Swift中的值类型主要是`结构体/枚举/元组`。

- 启动性能 - APP启动时值类型没有额外的消耗，class有一定额外的消耗。
- 运行时性能- 值类型不需要在堆上分配空间/额外的引用计数管理。更少的内存占用和更快的性能。
- 包大小 - 相比class，值类型不需要创建ObjC类对应的ro_data_t数据结构。


`提示`：class即使没有继承`NSObject`也会生成`ro_data_t`，里面包含了`ivars`属性信息。如果属性/方法申明为@objc还会生成对应的`方法列表`。

`提示`：struct无法代替class的一些场景：

- 1.需要使用继承调用super。
- 2.需要使用引用类型。
- 3.需要使用deinit。
- 4.需要在运行时动态转换一个实例的类型。


`提示`：不是所有struct都会保存在栈上，`部分数据大的struct也会保存在堆上`。

#### 集合元素使用值类型

集合元素使用值类型。因为NSArray并不支持值类型，编译器不需要处理可能需要桥接到NSArray的场景，可以移除部分消耗。
纯静态类型避免使用class

当class只包含静态方法/属性时，考虑使用enum代替class，因为class会生成更多的二进制代码。

不推荐

```swift
class Object {
    static var num: Int
    static func test() {}
}
```

推荐

```swift
enum Object {
    static var num: Int
    static func test() {}
}
```

`提示`：为什么用enum而不是struct，`因为struct会额外生成init方法`。

#### 值类型性能优化

考虑使用引用类型

值类型为了维持值语义，会在每次赋值/参数传递/修改时进行`复制`。虽然编译器本身会做一些优化，例如写时复制优化，在修改时减少复制频率，但是这仅针对于标准库提供的`集合和String`结构有效，对于自定义结构需要自己实现。对于参数传递编译器在一些场景会优化为直接传递引用的方式避免复制行为。

但是对于一些数据特别大的结构，同时需要频繁变更修改时也可以考虑使用引用类型实现。

#### 使用inout传递参数减少复制

虽然编译器本身会进行写时复制的优化，但是部分场景编译器无法处理。

不推荐

```swift
func append_one(_ a: [Int]) -> [Int] {
  var a = a
  a.append(1) // 无法被编译器优化，因为这时候有2个引用持有数组
  return a
}

var a = [1, 2, 3]
a = append_one(a)
```

推荐

直接使用inout传递参数

```swift
func append_one_in_place(a: inout [Int]) {
  a.append(1)
}

var a = [1, 2, 3]
append_one_in_place(&a)
```

#### 使用isKnownUniquelyReferenced实现写时复制

默认情况下结构体中包含引用类型，在修改时只会重新拷贝引用。但是我们希望`CustomData`具备值类型的特性，所以当修改时需要重新复制NSMutableData避免复用。但是复制操作本身是耗时操作，我们希望可以减少一些不必要的复制。

优化前

```swift
struct CustomData {
    fileprivate var _data: NSMutableData
    var _dataForWriting: NSMutableData {
        mutating get {
            _data = _data.mutableCopy() as! NSMutableData
            return _data
        }
    }
    init(_ data: NSData) {
        self._data = data.mutableCopy() as! NSMutableData
    }

    mutating func append(_ other: MyData) {
        _dataForWriting.append(other._data as Data)
    }
}

var buffer = CustomData(NSData())
for _ in 0..<5 {
    buffer.append(x) // 每一次调用都会复制
}
```

优化后

使用isKnownUniquelyReferenced检查如果是唯一引用不进行复制。

```
final class Box<A> {
    var unbox: A
    init(_ value: A) { self.unbox = value }
}

struct CustomData {
    fileprivate var _data: Box<NSMutableData>
    var _dataForWriting: NSMutableData {
        mutating get {
            // 检查引用是否唯一
            if !isKnownUniquelyReferenced(&_data) {
                _data = Box(_data.unbox.mutableCopy() as! NSMutableData)
            }
            return _data.unbox
        }
    }
    init(_ data: NSData) {
        self._data = Box(data.mutableCopy() as! NSMutableData)
    }
}

var buffer = CustomData(NSData())
for _ in 0..<5 {
    buffer.append(x) // 只会在第一次调用时进行复制
}
```

`提示`：对于ObjC类型`isKnownUniquelyReferenced`会直接返回false。

### 减少使用Objc特性

#### 避免使用Objc类型

尽可能避免在Swift中使用`NSString/NSArray/NSDictionary`等ObjC基础类型。以Dictionary为例，虽然Swift Runtime可以在NSArray和Array之间进行隐式桥接需要O(1)的时间。但是`字典当Key和Value既不是类也不是@objc协议时，需要对每个值进行桥接，可能会导致消耗O(n)时间`。

#### 减少添加@objc标识

@objc标识虽然不会强制使用消息转发的方式来调用方法/属性，但是他会默认ObjC是可见的会生成和ObjC一样的`ro_data_t`结构。


#### 避免使用@objcMembers

使用@objcMembers修饰的类，默认会`为类/属性/方法/扩展都加上@objc标识`。

```swift
@objcMembers class Object: NSObject {
}
```

`提示`：你也可以使用@nonobjc取消支持ObjC。

#### 避免继承NSObject

你只需要在需要使用NSObject特性时才需要继承，例如需要实现`UITableViewDataSource`相关协议。

#### 使用let变量/属性

##### 优化集合创建

集合不需要修改时，使用let修饰，编译器会优化创建集合的性能。例如针对let集合，编译器在创建时可以`分配更小的内存大小`。

##### 优化逃逸闭包

在Swift中，当捕获var变量时编译器需要生成一个在堆上的Box保存变量用于之后对于变量的读/写，同时需要额外的内存管理操作。如果是let变量，编译器可以保存值复制或引用，避免使用Box。


#### 避免使用大型struct使用class代替

大型struct通常是指`属性特别多并且嵌套类型很多`。目前swift编译器针对struct等值类型编译优化处理的并不好，会生成大量的`assignWithCopy`、`assignWithCopy`等copy相关方法，生成大量的二进制代码。使用class类型可以避免生成相关的copy方法。

`提示`：不要小看这部分二进制的影响，个人在日常项目中遇到过复杂的大型struct能生成`几百KB`的二进制代码。但是目前并没有好的方法去发现这类struct去做优化，只能通过相关工具去查看生成的二进制详细信息。希望官方可以早点优化。

#### 优先使用Encodable/Decodable协议代替Codable

因为实现`Encodable`和`Decodable`协议的结构，编译器在编译时会自动生成对应的`init(from decoder: Decoder)`和`encode(to: Encoder)`方法。`Codable`同时实现了`Encodable`和`Decodable`协议，但是大部分场景下我们只需要`encode`或`decode`能力，所以明确指定实现`Encodable`或`Decodable`协议可以减少生成对应的方法减少包体积。

`提示`：对于属性比较多的类型结构会产生很大的二进制代码，有兴趣可以用相关的工具看看生成的二进制文件。

####减少使用Equatable协议

因为实现`Equatable`协议的结构，编译器在编译时会自动生成对应的equal方法。`默认实现是针对所有字段进行比较会生成大量的代码`。所以当我们不需要实现==比较能力时不要实现Equatable或者对于属性特别多的类型也可以考虑重写Equatable协议，只针对部分属性进行比较，这样可以生成更少的代码减少包体积。

`提示`：对于属性特别多的类型也可以考虑重写Equatable协议，只针对部分属性进行比较，同时也可以提升性能。




























