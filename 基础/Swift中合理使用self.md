## Swift中如何正确使用self

在Swift中self是自身实例的一个特殊属性，大多数场景，self会出现在初始器或类、结构、枚举的方法中。

在Swift的[API](https://www.swift.org/documentation/api-design-guidelines/#clarity-over-brevity) 设计引导中提到了`Clarity is more important than brevity.`,即清晰明确比剪短更加重要。他暗示高效是第一位，他可以提升代码的可读性，但是可能影响代码的简洁性。

针对上面的设计引导，在self的使用中有两个相反的意见:

- 为了遵守设计原则，总是在访问实例属性时通过self进行访问
- 在不影响可读性的前提下，尽可能的省略self

实际上Swift鼓励我们在能省略self的情况下尽量省略self。问题在与什么场景下我们应该省略什么场景下我们需要强制使用。

### 访问实例的属性和方法

先来看下下面的例子:

```swift
struct Weather {
  
  let windSpeed: Int    // miles per hour
  let chanceOfRain: Int // percent
  init(windSpeed: Int, chanceOfRain: Int) {
    self.windSpeed = windSpeed
    self.chanceOfRain = chanceOfRain
  }
  func isDayForWalk() -> Bool {
    let comfortableWindSpeed = 5
    let acceptableChanceOfRain = 30
    return self.windSpeed <= comfortableWindSpeed
      && self.chanceOfRain <= acceptableChanceOfRain
  }
  
}
// A nice day for a walk
let niceWeather = Weather(windSpeed: 4, chanceOfRain: 25)
print(niceWeather.isDayForWalk()) // => true
```
上面的代码中我们在Weather结构体中，的`init`、`isDayForWalk`方法中使用self。
在上面的两个方法中，`self.windSpeed`和 `self.chanceOfRain`都是指当前Weather的实例。

虽然这个结构体看起来很好，但是很明显的可以感觉到self的冗余，因为无论是在`init`还是在`isDayForWalk`都是发生在结构体内部，我们是否可以去掉这个self呢？

这里建议，去掉`isDayForWalk`方法中的self，即

```swift
struct Weather {
  /* ... */
  func isDayForWalk() -> Bool {
    let comfortableWindSpeed = 5
    let acceptableChanceOfRain = 30
    return windSpeed <= comfortableWindSpeed
      && chanceOfRain <= acceptableChanceOfRain
  }
}
```

因为`isDayForWalk`方法一定是在Weather的实例下被调用，Swift允许我们省略掉self,直接访问`windSpeed`和`chanceOfRain`。

下面我们在来看下是否可以在init方法中去掉self:

```swift
init(windSpeed: Int, chanceOfRain: Int) {
  windSpeed = windSpeed
  chanceOfRain = chanceOfRain
}
```

由于在init方法中由于参数名和属性名是一样的，所以我们这里如果去掉了self,xcode会直接提示`Cannot assign to value: 'windSpeed' is a 'let' constant`

![initerrorwithoutself](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/initerrorwithoutself.png)

这时候self可以帮我们准确的区分是属性还是参数


### To be or not to be

实际上是否需要强制使用self，引发了很多的讨论` 使用self的有点很明显，他可以帮你很好的区分实例的属性还是你方法参数又或者是临时变量`,但是在我看来如果一个方法大到你需要是用self去区分属性还是其他的时候，这就说明你的方法过于庞大了。这很明显是违反单一职责原则的。使用self来协助你去区分当前对象的属性或者其他只是帮你暂时缓解繁冗的方法给你的来的问题，并不能解决问题。

因此当你的方法遵守了单一职责原则后，你的类或者结构体是非常简洁的，所以你不需要通过self来区分属性还是其他。

### 访问类型属性或者方法

在类方法中self指的是类型而非实例

```swift
struct Const {  
  static let minLimit = 0
  static let maxLimit = 250
  static func getLimitRange() -> ClosedRange<Int> {
    return self.minLimit...self.maxLimit    
  }
}
print(Const.getLimitRange()) // => 0...250 
```
这里如果我们省略掉self呢？

```swift
struct Const {
  static let minLimit = 0
  static let maxLimit = 250
  static func getLimitRange() -> ClosedRange<Int> {
    return minLimit...maxLimit
  }
}
print(Const.getLimitRange()) // => 0...250
```
这样实际也是没问题的，Swift很机制的推断出你要访问的是类型属性。

那么如果我们直接使用类型呢？

```swift
struct Const {
  static let minLimit = 0
  static let maxLimit = 250
  static func getLimitRange() -> ClosedRange<Int> {
    return Const.minLimit...Const.maxLimit
  }
}
print(Const.getLimitRange()) // => 0...250
```
这种方式也是没有问题的。

### 在block中访问实例属性或者方法

```swift
class Collection {
  var numbers: [Int]
  init(from numbers: [Int]) {
    self.numbers = numbers
  }
  func getAppendClosure() -> (Int) -> Void {
    return { 
      self.numbers.append($0)
    }            
  }
}
var primes = Collection(from: [2, 3, 5])
let appendToPrimes = primes.getAppendClosure()
appendToPrimes(7)
appendToPrimes(11)
print(primes.numbers) // => [2, 3, 5, 7, 11]
```
numbers是一个Int类型的数组，在getAppendClosure返回block中每次调用都会向numbers中追加一个Int整数。在block中我们使用self.numbers访问实例的属性。

但是由于闭包会持有block中的变量包括self,所以这么写可能会导致在self合block之间产生循环引用

我们在来看下下面这个场景:

```swift
class Person {
  let name: String
  lazy var sayMyName: () -> String = {
     return self.name   
  }
  
  init(withName name: String) {
    self.name = name
  }
  deinit {
    print("Person deinitialized")
  }
}
var leader: Person? 
leader = Person(withName: "John Connor")
if let leader = leader {
  print(leader.sayMyName())   
}
leader = nil
```
当leader被置为nil时，我们期待着deinit方法被调用，但实际上并没有

我们可以通过下面的方式解决

```swift
class Person {
  /* ... */
  lazy var sayMyName: () -> String = { 
     [unowned self] in
     return self.name   
  }
  /* ... */
  deinit {
    print("Person deinitialized")
  }
}
var leader: Person? 
leader = Person(withName: "John Connor")
if let leader = leader {
  print(leader.sayMyName())   
}
leader = nil
// => "Person deinitialized"
```

`注意`: 在block中添加`[unowned self]`,这表示self在block是不持有的引用，这样就破坏了引用循环，在leader被置为nil时，deinit方法就会像预期那样被调用。

这里建议阅读[这篇文章](https://www.uraimo.com/2016/10/27/unowned-or-weak-lifetime-and-performance/)了解更多.

重点：如果你要在一个闭包中使用self,无论何时都要考虑这是否会导致循环引用

### self单独使用

#### 方法链

当你想实现一个方法链条调用时，self作为返回值的方式将常常用到

例如下面这个例子:

```swift
class Stack<Element> {
  fileprivate var elements = [Element]()
  @discardableResult
  func push(_ element: Element) -> Stack {
    elements.append(element)
    return self
  }
  func pop() -> Element? {
    return elements.popLast()
  }
  
  func printElements() {
    print(elements)
  }
}
var numbers = Stack<Int>()
numbers
  .push(8)
  .push(10)
  .push(2)
numbers.printElements() // => [8, 10, 2]
```

`push`方法的返回值是self，为的就是你可以在使用时通过点语法多次想栈中push元素。

#### 枚举

在进行枚举的判断时我们也会经常用到self

```swift
enum Activity {
  case sleep
  case code
  case learn
  case procrastinate
  func getOccupation() -> String {
    switch self {
      case .sleep:
        return "Sleeping"
      case .code:
        return "Coding"
      case .learn:
        return "Reading a book"
      default:
        return "Enjoying laziness"
    }    
  }
}
let improving = Activity.learn
print(improving.getOccupation()) // => "Reading a book"
```

#### 新的结构体实例

我们之前看到过mutating关键词，在这个关键词修饰的方法中可以修改self

```swift
struct Movement {
  var speed: Int
  mutating func stop() {
    self = Movement(speed: 0)
  }
}
var run = Movement(speed: 20)
print(run.speed) // => 20
run.stop() 
print(run.speed) // => 0
```

通过上面这种方式我们可以给self赋值一个全新的值。

### 总结

- self既可以表示类的实例也可以表示类型。

- Swift允许在访问实例属性的时候省略掉self，这里也建议能省则省

- 当方法的参数和实例的属性重名时需要使用self指定，否则编译器会报错


### 参考文档

[How to Use Correctly 'self' Keyword in Swift](https://dmitripavlutin.com/how-to-use-correctly-self-keyword-in-swift/)

