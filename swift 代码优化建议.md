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































