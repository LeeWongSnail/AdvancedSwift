## 属性

在Swift中属性可以分为存储属性和计算属性，计算属性可以在类、结构体、枚举中定义，存储属性只能在类和结构体中定义。

### 存储属性

存储属性要么是变量存储属性（由var关键字引入）要么是常量存储属性（由let关键字引入）

```swift
struct FixedLengthRange {
    var firstValue: Int
    let length: Int
}
```

#### 常量结构体实例的存储属性

如果你创建了一个结构体的实例并且把这个实例赋给常量，你不能修改这个实例的属性，即使是声明为变量的属性：

```swift
let rangeOfFourItems = FixedLengthRange(firstValue: 0, length: 4)
// this range represents integer values 0, 1, 2, and 3
rangeOfFourItems.firstValue = 6
// this will report an error, even though firstValue is a variable property
```

`这是由于结构体是值类型。当一个值类型的实例被标记为常量时，该实例的其他属性也均为常量。`

#### 延迟存储属性(Lazy)

延迟存储属性的初始值在其第一次使用时才进行计算。你可以通过在其声明前标注 lazy 修饰语来表示一个延迟存储属性。

`你必须把延迟存储属性声明为变量（使用var关键字），因为它的初始值可能在实例初始化完成之前无法取得。常量属性则必须在初始化完成之前有值，因此不能声明为延迟。`

使用场景:

- 属性的初始值可能依赖于某些外部因素, 当这些外部因素的值只有在实例的初始化完成后才能得到时

- 而当属性的初始值需要执行复杂或代价高昂的配置才能获得，你又想要在需要时才执行

```swift
class DataImporter {
    
    //DataImporter is a class to import data from an external file.
    //The class is assumed to take a non-trivial amount of time to initialize.
    
    var fileName = "data.txt"
    // the DataImporter class would provide data importing functionality here
}
 
class DataManager {
    lazy var importer = DataImporter()
    var data = [String]()
    // the DataManager class would provide data management functionality here
}
 
let manager = DataManager()
manager.data.append("Some data")
manager.data.append("Some more data")
// the DataImporter instance for the importer property has not yet been created
```

DataManager类中的importer被定义为lazy,因此知道上面的代码执行完成，importer都没有被初始化，因为没有被使用过。

```swift
print(manager.importer.fileName)
// the DataImporter instance for the importer property has now been created
// prints "data.txt"
```

注意: `如果被标记为 lazy 修饰符的属性同时被多个线程访问并且属性还没有被初始化，则无法保证属性只初始化一次。`

#### 存储属性与实例变量

在OC中每一个property都对应着一个ivar。

Swift 属性没有与之相对应的实例变量，并且属性的后备存储不能被直接访问。


### 计算属性

计算属性实际上并不存储值，他们提供一个读取器和一个可选的设置器来间接得到和设置其他的属性和值。

```swift
struct Point {
    var x = 0.0, y = 0.0
}
struct Size {
    var width = 0.0, height = 0.0
}
struct Rect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + (size.width / 2)
            let centerY = origin.y + (size.height / 2)
            return Point(x: centerX, y: centerY)
        }
        set(newCenter) {
            origin.x = newCenter.x - (size.width / 2)
            origin.y = newCenter.y - (size.height / 2)
        }
    }
}
var square = Rect(origin: Point(x: 0.0, y: 0.0),
    size: Size(width: 10.0, height: 10.0))
let initialSquareCenter = square.center
square.center = Point(x: 15.0, y: 15.0)
print("square.origin is now at (\(square.origin.x), \(square.origin.y))")
// prints "square.origin is now at (10.0, 10.0)"
```

这里我们看到Rect中的center就是一个计算属性，我们可以获取甚至修改他的值，但是Rect中实际并没有存储center的值，center只是由现有的字段计算得出。修改的时候也是修改他所依赖的字段(origin,size)。

`你必须用 var 关键字定义计算属性(包括只读计算属性)为变量属性，因为它们的值不是固定的。 let 关键字只用于常量属性，用于明确那些值一旦作为实例初始化就不能更改。`

#### 简化设置其声明

我们可以简化计算属性的set方法: `省略掉newValue`

```swift
struct AlternativeRect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + (size.width / 2)
            let centerY = origin.y + (size.height / 2)
            return Point(x: centerX, y: centerY)
        }
        set {
            origin.x = newValue.x - (size.width / 2)
            origin.y = newValue.y - (size.height / 2)
        }
    }
}
```

#### 简化读取器声明

```swift
var center: Point {
    get {
        Point(x: origin.x + (size.width / 2),
              y: origin.y + (size.height / 2))
    }
    set {
        origin.x = newValue.x - (size.width / 2)
        origin.y = newValue.y - (size.height / 2)
    }
}
```

#### 只读计算属性

即只有get方法的属性

```swift
struct Cuboid {
    var width = 0.0, height = 0.0, depth = 0.0
    var volume: Double {
        return width * height * depth
    }
}
```

### 属性观察者

属性观察者会观察并对属性值的变化做出回应。每当一个属性的值被设置时，属性观察者都会被调用，`即使这个值与该属性当前的值相同`。

一般我们可以在下面几个场景里使用属性观察者

```swift
你定义的存储属性
你继承的存储属性
你继承的计算属性
```
对于自己定义的计算属性，则不需要使用观察者，因为你会自己实现get/set方法，直接在对应的方法中进行操作即可。

- willSet 会在该值被存储之前被调用, 收到的参数为newValue
- didSet 会在一个新值被存储后被调用, 收到的参数是oldValue

`父类属性的 willSet 和 didSet 观察者会在子类初始化器中设置时被调用。它们不会在类的父类初始化器调用中设置其自身属性时被调用。`


```swift
class StepCounter {
    var totalSteps: Int = 0 {
        willSet(newTotalSteps) {
            print("About to set totalSteps to \(newTotalSteps)")
        }
        didSet {
            if totalSteps > oldValue  {
                print("Added \(totalSteps - oldValue) steps")
            }
        }
    }
}
let stepCounter = StepCounter()
stepCounter.totalSteps = 200
// About to set totalSteps to 200
// Added 200 steps
stepCounter.totalSteps = 360
// About to set totalSteps to 360
// Added 160 steps
stepCounter.totalSteps = 896
// About to set totalSteps to 896
// Added 536 steps
```

注意: `如果你以输入输出形式参数传一个拥有观察者的属性给函数，且在函数中修改了属性， willSet 和 didSet 观察者一定会被调用。这是由于输入输出形式参数的*拷贝入拷贝出*存储模型导致的：值一定会在函数结束后写回属性。更多关于输入输出形式参数行为的讨论，参见输入输出形式参数。`

### 属性包装

属性包装给代码之间添加了一层分离层，它用来管理属性如何存储数据以及代码如何定义属性。

```swift
@propertyWrapper
struct TwelveOrLess {
    private var number = 0
    var wrappedValue: Int {
        get { return number }
        set { number = min(newValue, 12) }
    }
}
```

我们先不看`@propertyWrapper`,单看TwelveOrLess结构体，内部有一个private的number属性，提供了一个计算熟属性wrappedValue。外部如果想要修改number就只能通过wrappedValue,这就是属性的包装。更重要的是TwelveOrLess可以保证number的大小是小于12的。

```swift
struct SmallRectangle {
    @TwelveOrLess var height: Int
    @TwelveOrLess var width: Int
}
```

下面我们来验证下:

```swift
var rectangle = SmallRectangle()
print(rectangle.height)
// Prints "0"
 
rectangle.height = 10
print(rectangle.height)
// Prints "10"
 
rectangle.height = 24
print(rectangle.height)
// Prints "12"
```

实际上属性包装是帮我们进行了封装，下面是系统帮我们实现的代码：

```swift
struct SmallRectangle {
    private var _height = TwelveOrLess()
    private var _width = TwelveOrLess()
    var height: Int {
        get { return _height.wrappedValue }
        set { _height.wrappedValue = newValue }
    }
    var width: Int {
        get { return _width.wrappedValue }
        set { _width.wrappedValue = newValue }
    }
}
```

详情可以查看我的另外一片文章[Property Wrapper]()

### 全局变量和局部变量

全局变量是定义在任何函数、方法、闭包或者类型环境之外的变量。局部变量是定义在函数、方法或者闭包环境之中的变量。

全局常量和变量永远是延迟计算的，与延迟存储属性有着相同的行为。不同于延迟存储属性，全局常量和变量不需要标记 lazy 修饰符。


### 类型属性



