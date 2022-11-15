## Subscript

![subscriptdefine](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/subscriptdefine.png)

下标是元素访问的一个强大统一的接口，可以覆盖写、重载，可以定义多维下标。

下面是Swift中Dictionary中实现的[subscript](https://github.com/apple/swift/blob/main/stdlib/public/core/Dictionary.swift)方法

```swift
@inlinable
public subscript(key: Key) -> Value? {
  get {
    return _variant.lookup(key)
  }
  set(newValue) {
    if let x = newValue {
      _variant.setValue(x, forKey: key)
    } else {
      removeValue(forKey: key)
    }
  }
  _modify {
    defer { _fixLifetime(self) }
    yield &_variant[key]
  }
}
```

我们都知道Subscript对于集合类型，非常方便快捷。而在Swift中小标的使用范围得到了进一步的扩展，我们可以给自定义的类添加下标方法，并访问其中的元素。

同时对于一个类型，我们可以定义多个`subscripts`，具体使用哪一个下标，由传入下标`subscript`的索引类型决定。可以定义多为下标

例如 `subscript(index: Int, name: String) -> Int`




### 下标语法

使用关键字 `subscript` 开始声明下标，之后是一个或是多个输入参数+一个返回类型

```swift
subscript(index: Int) -> Int {
    get {
        // Return an appropriate subscript value here.
    }
    set(newValue) {
        // Perform a suitable setting action here.
    }
}
```

根据其中方法实现的不同，`subscript`又可以分为`只读subscript`和`读写subscript`

```swift
subscript(index: Int) -> Int {
    // Return an appropriate subscript value here.
}
```

### 下标选项

下标可以有任意个数的参数，也可以是任意类型，也可以返回任何类型。但下标的参数是单向的，只能输入不能输出(函数的参数是可以inout的)

类或者结果提可以提供多个`subscript`的实现，具体哪一个`subscript`会被调用取决于被调用时`[]`号内值的类型。这种多个`subscript`定义的方式也被成为`subscript overloading.`

```swift
struct Matrix {
    let rows: Int, columns: Int
    var grid: [Double]
    init(rows: Int, columns: Int) {
        self.rows = rows
        self.columns = columns
        grid = Array(repeating: 0.0, count: rows * columns)
    }
    func indexIsValid(row: Int, column: Int) -> Bool {
        return row >= 0 && row < rows && column >= 0 && column < columns
    }
    subscript(row: Int, column: Int) -> Double {
        get {
            assert(indexIsValid(row: row, column: column), "Index out of range")
            return grid[(row * columns) + column]
        }
        set {
            assert(indexIsValid(row: row, column: column), "Index out of range")
            grid[(row * columns) + column] = newValue
        }
    }
}
```
`Matrix`提供了一个参数为`rows`和`columns`的初始化方法，来创建一个足够大的array来容纳要放置的元素，每个元素的默认值为0.0.

我们可以通过下面这个方法来创建一个`matrix`实例

```swift
var matrix = Matrix(rows: 2, columns: 2)
```
这个实例创建了一个2行2列的矩阵，

![subscriptMatrix_row](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/subscriptMatrix_row.png)

我们可以通过下面这种方式给Matrix赋值

```swift
matrix[0, 1] = 1.5
matrix[1, 0] = 3.2
```
赋值后:

![subscriptMatrix_afterassign](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/subscriptMatrix_afterassign.png)

在`matrix`的`setter`和`getter`方法前我们都进行了`assert`,判断下标是否合法

`assert(indexIsValid(row: row, column: column), "Index out of range")`

如果我们要通过下标获取或者设置`matrix`时，会提示我们错误

```swift
let someValue = matrix[2, 2]
// This triggers an assert, because [2, 2] is outside of the matrix bounds.
```

### 类型下标

我们上面讲述的都是对某个结构体或者类的实例添加下标方法(方法都是实例方法)，那么我们是否可以给类型添加下标方法呢？

答案是肯定的

```swift
enum Planet: Int {
    case mercury = 1, venus, earth, mars, jupiter, saturn, uranus, neptune
    static subscript(n: Int) -> Planet {
        return Planet(rawValue: n)!
    }
}
let mars = Planet[4]
print(mars)
```

这里我们也可以用`class`代替`static`。


### 自定义下标

假设我们有下面这个类来存储我们每天要吃什么

```swift
class DailyMeal {
    enum MealTime {
        case Breakfast
        case Lunch
        case Dinner
    }
    
    var meals: [MealTime : String] = [:]
}
```
我们希望通过下面这个方式可以获取我们要吃的东西:

```swift
let monday = DailyMeal()

monday.meals[.Breakfast] = "Toast"

if let someMeal = monday.meals[.Breakfast] {
    print(someMeal)
}
```

那么我们只需要实现这个下标的方法:

```swift
subscript(requestedMeal: MealTime) -> String? {
    get {
        return meals[requestedMeal]
    }
    set(newMealName) {
        meals[requestedMeal] = newMealName
    }
}
```
感觉这根Swift中的计算属性非常相似。只是我们使用的是`subscript`关键字。

添加上面的实现后，我们期望的调用方法就完全没问题了输出的结果为`”Toast“`。

而且，我们可以在`get`方法中添加默认返回值

```swift
subscript(requestedMeal: MealTime) -> String {
    get {
        if let thisMeal = meals[requestedMeal] {
            return thisMeal
        } else {
            return "Ramen"
        }
    }

    set(newMealName) {
        meals[requestedMeal] = newMealName
    }
}
```
当我们在调用

```swift
let monday = DailyMeal()

monday[.Lunch] = "Pizza"

print(monday[.Lunch])         
//"Pizza"

print(monday[.Dinner])        
//"Ramen"
```

### 只读subscript

我们上面提到，当只实现`get`方法时，`subscript`就会默认是一个只读的`subscript`。其实如果是一个`只读`的`subscript`。我们可以直接省略get。

例如

```swift
struct FactorialGenerator {
    subscript(n: Int) -> Int {
        var result = 1

        if n > 0 {
            for value in 1...n {
                result *= value
            }
        }

        return result
    }
}
```

我们有一个`阶乘`的结构体，传入对应参数后返回对应参数的阶乘结果。

```swift
let factorial = FactorialGenerator()

print("Five factorial is equal to \(factorial[5]).")
//"Five factorial is equal to 120."

print("Ten Factorial is equal to \(factorial[10]).")
//"Ten Factorial is equal to 3628800."
```

其实关于subscript的使用我们可以参考[SwiftJSON](https://github.com/lingoer/SwiftyJSON)这个库，其中`subscript`使用可以作为一个范例

```swift
public subscript(path: [SubscriptType]) -> JSON {
    get {
        if path.count == 0 {
            return JSON.nullJSON
        }
        
        var next = self
        for sub in path {
            next = next[sub:sub]
        }
        return next
    }
    set {
        
        switch path.count {
        case 0: return
        case 1: self[sub:path[0]] = newValue
        default:
            var last = newValue
            var newPath = path
            newPath.removeLast()
            for sub in path.reverse() {
                var previousLast = self[newPath]
                previousLast[sub:sub] = last
                last = previousLast
                if newPath.count <= 1 {
                    break
                }
                newPath.removeLast()
            }
            self[sub:newPath[0]] = last
        }
    }
}
```

```swift
//With an array like path to the element
let path = [1,"list",2,"name"]
let name = json[path].string 
//Just the same
let name = json[1]["list"][2]["name"].string
```

### 参考文献

[Subscripts](https://docs.swift.org/swift-book/LanguageGuide/Subscripts.html)

[Subscript Syntax in Swift Explained](https://www.appypie.com/subscript-syntax-swift)

