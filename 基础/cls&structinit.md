## 初始化器委托

初始化是为类、结构体或者枚举`准备实例`的过程。这个过需要给实例里的每一个存储属性设置一个初始值并且在新实例可以使用之前执行任何其他所必须的配置或初始化。

不同于 Objective-C 的初始化器，Swift 初始化器不返回值。

### 什么是初始化器委托

初始化委托器在值类型和引用类型中是不一样的

- 值类型(结构体和枚举)不支持继承，所以他它们的初始化器委托的过程相当简单，因为它们只能提供它们自己为另一个初始化器委托。

- 类有额外的责任来确保它们继承的所有存储属性在初始化期间都分配了一个合适的值。

初始化器`可以调用其他初始化器来执行部分实例的初始化`。这个过程，就是所谓的初始化器委托，避免了多个初始化器里冗余代码。

### 值类型的初始化器委托

```swift
struct Size {
    var width = 0.0, height = 0.0
}
struct Point {
    var x = 0.0, y = 0.0
}
struct Rect {
    var origin = Point()
    var size = Size()
    init() {}
    init(origin: Point, size: Size) {
        self.origin = origin
        self.size = size
    }
    init(center: Point, size: Size) {
        let originX = center.x - (size.width / 2)
        let originY = center.y - (size.height / 2)
        self.init(origin: Point(x: originX, y: originY), size: size)
    }
}
```
上面的Rect结构体共有三种初始化方法:

- init() 与默认初始化方法一致内部不会进行任何操作 origin和size都会默认为0
- init(origin: Point, size: Size) 与成员初始化器相同
- init(center: Point, size: Size) 通过外部传入的size和center计算出origin，然后委托init(origin: Point, size: Size)方法来初始化Rect

这种情况下，外部可以使用任何一种方法去初始化Rect这个类。

### 引用类型的初始化器

所有类的存储属性(包括从他父类继承的所有属性)都必须在`初始化期间分配初始值`

swift为引用类型定义了`两种初始化器`以确保所有的存储属性接受一个初始值，这些就是所谓的`指定初始化器`和`便捷初始化器`。

#### 指定初始化器和便捷初始化器

指定初始化器:

```swift
init(parameters) {
    statements
}
```

便捷初始化器:

```swift
convenience init(parameters) {
    statements
}
```

二者的区别和联系:

- 指定初始化器是主要的初始化器，而便捷初始化器是次要的，是可有可无的
- 指定初始化器是初始化开始并持续初始化过程到父类链的传送节点
- 每个类都至少有一个指定初始化器，在某些情况可以通过从父类继承一个或多个指定初始化器来满足
- 便捷初始化器可以调用指定初始化器来给指定的初始化器设置默认参数


#### 引用类型初始化器委托

swift在初始化器之间的委托调用有下面三个原则:

- 指定初始化器必须从他的直系父类调用指定初始化器
- 便捷初始化器必须从相同的类里调用另一个初始化器
- 便捷初始化器最终必须调用一个指定初始化器

总结下来就是: 指定初始化器必须总是向上委托，便捷初始化器必须总是横向委托，具体如下图:

![referType_initializerDelegation](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/referType_initializerDelegation.png)

#### 两段式初始化

swift的类初始化是一个两段式的过程

swift执行四种安全检查来确保两段式初始化顺利完成:

##### 安全检查一

指定初始化器必须保证在向上委托给父类初始化器之前，其所在类引入的所有属性都要初始化完成。

我们可以理解为： 在调用父类的初始化之前，子类的所有存储属性必须初始化完成

##### 安全检查二

指定初始化器必须先向上委托父类初始化器，然后才能为继承的属性设置新值。如果不这样做，指定初始化器赋予的新值将被父类中的初始化器所覆盖。

我们可以理解为： 在调用父类的初始化方法之前，子类不可以修改父类的存储属性，否则即使修改也会被父类覆盖。

##### 安全检查三

便捷初始化器必须先委托同类中的其它初始化器，然后再为任意属性赋新值（包括同类里定义的属性）。如果没这么做，便捷构初始化器赋予的新值将被自己类中其它指定初始化器所覆盖。

由于便捷初始化器只能横向依赖且最终肯定会调用到指定初始化器，因此必须先让当前类和父类的初始化都完成后才可以进行当前类的修改。否则也可能会被覆盖。

##### 安全检查四

初始化器在第一阶段初始化完成之前，不能调用任何实例方法、不能读取任何实例属性的值，也不能引用 self 作为值。

第一阶段完成前(即所有存储属性分配初始值前) 不能够使用功能self,包括获取属性或者调用实例方法，直到第一阶段结束类实例才完全合法。属性只能被读取，方法也只能被调用，直到第一阶段结束的时候，这个类实例才被看做是合法的。

#### 两段式

##### 阶段一

* 指定或便捷初始化器在类中被调用；
* 为这个类的新实例分配内存。内存还没有被初始化；
* 这个类的指定初始化器确保所有由此类引入的存储属性都有一个值。现在这些存储属性的内存被初始化了；
* 指定初始化器上交父类的初始化器为其存储属性执行相同的任务；
* 这个调用父类初始化器的过程将沿着初始化器链一直向上进行，直到到达初始化器链的最顶部；
* 一旦达了初始化器链的最顶部，在链顶部的类确保所有的存储属性都有一个值，此实例的内存被认为完全初始化了，此时第一阶段完成。

##### 阶段二

* 从顶部初始化器往下，链中的每一个指定初始化器都有机会进一步定制实例。初始化器现在能够访问 self 并且可以修改它的属性，调用它的实例方法等等；
* 最终，链中任何便捷初始化器都有机会定制实例以及使用 slef 。

##### 图示

![referType_twoPhaseInitialization](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/referType_twoPhaseInitialization.png)

在这个例子中，初始化过程从一个子类的便捷初始化器开始。这个便捷初始化器还不能修改任何属性。它委托给了同一类里的指定初始化器。

指定初始化器确保所有的子类属性都有值，如安全检查1。然后它调用父类的指定初始化器来沿着初始化器链一直往上完成父类的初始化过程。

父类的指定初始化器确保所有的父类属性都有值。由于没有更多的父类来初始化，也就不需要更多的委托。

一旦父类中所有属性都有初始值，它的内存就被认为完全初始化了，第一阶段完成。

下图是相同的初始化过程在第二阶段的样子：

![referType_twoPhaseInitialization02_2x](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/referType_twoPhaseInitialization02_2x.png)

现在父类的指定初始化器有机会来定制更多实例(尽管没有这种必要)。

一旦父类的指定初始化器完成了调用，子类的指定初始化器就可以执行额外的定制(同样，尽管没有这种必要)。

最后，一旦子类的指定初始化器完成，最初调用的便捷初始化器将会执行额外的定制操作。

#### 初始化器的继承和重写

不像在 Objective-C 中的子类，`Swift 的子类不会默认继承父类的初始化器`。

Swift 的这种机制`防止父类的简单初始化器被一个更专用的子类继承`并被用来创建一个没有完全或错误初始化的新实例的情况发生。


在查看下面内容之前，我们务必要了解清楚下面几个初始化器的含义:

- 指定初始化器 初始化所有那个类引用的属性并且调用合适的父类初始化器来继续这个初始化过程给父类链 一个类通常只有一个 关键词 super
- 便捷初始化器 convenience 不能独立初始化所有属性，需要调用指定初始化器

- 默认初始化器 init() 默认初始化器(如果可用的话)总是类的指定初始化器
- 成员初始化器 Size(width: 2.0, height: 2.0)
- 自定义初始化器 


我们先看例子:

```swift
class Vehicle {
    var numberOfWheels = 0
    var description: String {
        return "\(numberOfWheels) wheel(s)"
    }
}
```
Vehicle 类只为它的存储属性提供了默认值，并且没有提供自定义的初始化器。它会自动收到一个`默认初始化器`。

```swift
let vehicle = Vehicle()
print("Vehicle: \(vehicle.description)")
// Vehicle: 0 wheel(s)
```
面的例子定义了一个名为 Bicycle 继承自 Vehicle 的子类：

```swift
class Bicycle: Vehicle {
    override init() {
        super.init()
        numberOfWheels = 2
    }
}
```
子类 Bicycle 定义了一个自定义初始化器 init() 。这个指定初始化器和 Bicycle 的父类的指定初始化器相匹配，所以 Bicycle 中的指定初始化器需要带上 override 修饰符。

看了上面的例子之后我们在来理解下下面这段文字:

- 当你写的子类初始化器匹配父类指定初始化器的时候，你实际上可以重写那个初始化器。因此，在子类的初始化器定义之前你必须写 override 修饰符。即使是自动提供的默认初始化器你也可以重写。
这一个便如上面Bicycle重写的init方法

- 如果你写了一个匹配父类便捷初始化器的子类初始化器，父类的便捷初始化器将永远不会通过你的子类直接调用。因此，你的子类不能(严格来讲)提供父类初始化器的重写。当提供一个匹配的父类便捷初始化器的实现时，你不用写 override 修饰符。

```swift
class FatherClass {
    var name: String
    init(name: String) {
        self.name = name
    }
    convenience init() {
        self.init(name: "Lee")
    }
}

class SonClass: FatherClass {
    init() {
        super.init(name: "Wong")
    }
}
```

#### 初始化器的自动继承

##### 规则一

如果你的子类没有定义任何指定初始化器，它会自动继承父类所有的指定初始化器

##### 规则二

如果你的子类提供了所有父类指定初始化器的实现——要么是通过规则1继承来的，要么通过在定义中提供自定义实现的——那么它自动继承所有的父类便捷初始化器。

##### 验证

这个例子定义了 Food ， RecipeIngredient 和 ShoppingListItem 三个类的层级关系，并演示了它们的继承关系是如何相互作用的。

```swift
class Food {
    var name: String
    init(name: String) {
        self.name = name
    }
    convenience init() {
        self.init(name: "[Unnamed]")
    }
}
```

下面是Food的初始化链

![initializersExample01_2x]()

这里init(name: String)是指定初始化器 因为没有父类不需要调用super

```swift
let namedMeat = Food(name: "Bacon")
// namedMeat's name is "Bacon"
```

这里convenience init()是一个便捷初始化器，通过将"Unnamed"传给指定初始化器来初始化类

```swift
let mysteryMeat = Food()
// mysteryMeat's name is "[Unnamed]"
```

第二个类是Food的子类RecipeIngredient

```swift
class RecipeIngredient: Food {
    var quantity: Int
    init(name: String, quantity: Int) {
        self.quantity = quantity
        super.init(name: name)
    }
    override convenience init(name: String) {
        self.init(name: name, quantity: 1)
    }
}
```
引入了一个quantity标识调味品的数量

下面是RecipeIngredient的初始化关系链

![RecipeIngredient]()

- init(name: String, quantity: Int) 是指定初始化器

- init(name: String) 是一个便捷初始化器，因为与父类的指定初始化器名称和形参相同因此必须添加override修饰符。
- RecipeIngredient 类为所有的父类指定初始化器提供了实现。因此， RecipeIngredient 类也自动继承了父类所有的便捷初始化器。

因此RecipeIngredient有三种初始化器可以使用：

```swift
let oneMysteryItem = RecipeIngredient()
let oneBacon = RecipeIngredient(name: "Bacon")
let sixEggs = RecipeIngredient(name: "Eggs", quantity: 6)
```
`注意`: 这里从父类继承的convenience init()初始化器内部调用的是子类RecipeIngredient中重载的override convenience init(name: String)而非init(name: String),因此 这种方式可以初始化所有存储属性的值，其中 `quantity=1,name=[Unnamed]`。

类层级中第三个也是最后一个类是 RecipeIngredient 的子类，叫做 ShoppingListItem

```swift
class ShoppingListItem: RecipeIngredient {
    var purchased = false
    var description: String {
        var output = "\(quantity) x \(name)"
        output += purchased ? " ✔" : " ✘"
        return output
    }
}
```
ShoppingListItem 没有定义初始化器来给 purchased 一个初始值，这是因为任何添加到购物单（这里的模型）的项的初始状态总是未购买。

下图展示了三个类的初始化链：

![initializersExample03_2x]()

你可以使用全部三个继承来的初始化器来创建 ShoppingListItem 的新实例：

```swift
var breakfastList = [
    ShoppingListItem(),
    ShoppingListItem(name: "Bacon"),
    ShoppingListItem(name: "Eggs", quantity: 6),
]
breakfastList[0].name = "Orange juice"
breakfastList[0].purchased = true
for item in breakfastList {
    print(item.description)
}
// 1 x Orange juice ✔
// 1 x Bacon ✘
// 6 x Eggs ✘
```


### 可失败初始化器

在类、结构体或枚举中定义一个或多个可失败的初始化器。通过在 init 关键字后面添加问号`init?`来写

通过在可失败初始化器写 return nil 语句，来表明可失败初始化器在何种情况下会触发初始化失败。虽然你写了 return nil 来触发初始化失败，但是你不能使用 return 关键字来表示初始化成功了。

例如Int类型的这个方法

```swift
    public init?(exactly source: Float16)
```

```swift
let wholeNumber: Double = 12345.0
let pi = 3.14159
 
if let valueMaintained = Int(exactly: wholeNumber) {
    print("\(wholeNumber) conversion to int maintains value")
}
// Prints "12345.0 conversion to int maintains value"
 
let valueChanged = Int(exactly: pi)
// valueChanged is of type Int?, not Int
 
if valueChanged == nil {
    print("\(pi) conversion to int does not maintain value")
}
// Prints "3.14159 conversion to int does not maintain value"
```

wholeNumber可以调用成功，但是pi确失败了。

例如对于我们的自定义类

```swift
struct Animal {
    let species: String
    init?(species: String) {
        if species.isEmpty { return nil }
        self.species = species
    }
}
```
我们也可以通过这种方式来添加一个可是白的初始化器。

下面我们再来看看其他几种类型

#### 枚举的可失败初始化器

```swift
enum TemperatureUnit {
    case Kelvin, Celsius, Fahrenheit
    init?(symbol: Character) {
        switch symbol {
        case "K":
            self = .Kelvin
        case "C":
            self = .Celsius
        case "F":
            self = .Fahrenheit
        default:
            return nil
        }
    }
}
```

#### 带有原始值枚举的可失败初始化器

```swift
enum TemperatureUnit: Character {
    case Kelvin = "K", Celsius = "C", Fahrenheit = "F"
}
 
let fahrenheitUnit = TemperatureUnit(rawValue: "F")
if fahrenheitUnit != nil {
    print("This is a defined temperature unit, so initialization succeeded.")
}
// prints "This is a defined temperature unit, so initialization succeeded."
 
let unknownUnit = TemperatureUnit(rawValue: "X")
if unknownUnit == nil {
    print("This is not a defined temperature unit, so initialization failed.")
}
// prints "This is not a defined temperature unit, so initialization failed."
````

#### 初始化失败的传递

```swift
class Product {
    let name: String
    init?(name: String) {
        if name.isEmpty { return nil }
        self.name = name
    }
}
 
class CartItem: Product {
    let quantity: Int
    init?(name: String, quantity: Int) {
        if quantity < 1 { return nil }
        self.quantity = quantity
        super.init(name: name)
    }
}
```

#### 重写可失败初始化器

可以在子类里重写父类的可失败初始化器。。或者，你可以用子类的非可失败初始化器来重写父类可失败初始化器。这样允许你定义一个初始化不会失败的子类，尽管父类的初始化允许失败。

```swift
class Document {
    var name: String?
    // this initializer creates a document with a nil name value
    init() {}
    // this initializer creates a document with a non-empty name value
    init?(name: String) {
        self.name = name
        if name.isEmpty { return nil }
    }
}
```

他有一个子类AutomaticallyNamedDocument

```swift
class AutomaticallyNamedDocument: Document {
    override init() {
        super.init()
        self.name = "[Untitled]"
    }
    override init(name: String) {
        super.init()
        if name.isEmpty {
            self.name = "[Untitled]"
        } else {
            self.name = name
        }
    }
}
```

`重点`: 你可以用一个非可失败初始化器重写一个可失败初始化器，但反过来是不行的。

你可以在初始化器里`使用强解包`来从父类调用一个`可失败初始化器`作为`子类``非可失败初始化器`的一部分。

例如，下边的 UntitledDocument 子类将总是被命名为 "[Untitled]" ，并且在初始化期间它使用了父类的可失败 init(name:) 初始化器：

```swift
class UntitledDocument: Document {
    override init() {
        super.init(name: "[Untitled]")!
    }
}
```

### 必要初始化器

在类的初始化器前添加 required  修饰符来表明所有该类的子类都必须实现该初始化器：

```swift
class SomeClass {
    required init() {
        // initializer implementation goes here
    }
}
```
当子类重写父类的必要初始化器时，必须在子类的初始化器前同样添加 required 修饰符以确保当其它类继承该子类时，该初始化器同为必要初始化器。在重写父类的必要初始化器时，不需要添加 override 修饰符：

```
class SomeSubclass: SomeClass {
    required init() {
        // subclass implementation of the required initializer goes here
    }
}
```

### 通过闭包和函数来设置属性的默认值

```swift
class SomeClass {
    let someProperty: SomeType = {
        // create a default value for someProperty inside this closure
        // someValue must be of the same type as SomeType
        return someValue
    }()
}
```
注意闭包花括号的结尾跟一个没有参数的圆括号。这是告诉 Swift 立即执行闭包。如果你忽略了这对圆括号，你就会把闭包作为值赋给了属性，并且不会返回闭包的值。

`重点`: 如果你使用了闭包来初始化属性，请记住闭包执行的时候，实例的其他部分还没有被初始化。这就意味着你不能在闭包里读取任何其他的属性值，即使这些属性有默认值。你也不能使用隐式 self 属性，或者调用实例的方法。


### 参考文献

[初始化](https://www.cnswift.org/initialization/#spl-18)

