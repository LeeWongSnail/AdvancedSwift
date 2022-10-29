## 访问控制

访问控制可以限定其它源文件或模块对你的代码的访问。这个特性可以让你隐藏代码的实现细节，并且能提供一个接口来让别人访问和使用你的代码。

---

### 访问级别

通过`public`和`final`限制模块外使用`class`不能被继承和重写。

访问控制可以限定其它源文件或模块对你的代码的访问

可以明确地给`单个类型`（类、结构体、枚举）设置访问级别，也可以给这些类型的`属性、方法、构造器、下标`等设置访问级别。

- `open` 和 `public` 级别可以让实体被同一模块源文件中的所有实体访问，在模块外也可以通过导入该模块来访问源文件里的所有实体。

	`open` 为最高访问级别(限制最少)

	`open` 只能作用于类和类的成员，它和 `public` 的区别主要在于 `open` 限定的类和成员能够在模块外能被继承和重写

- `internal` 级别让实体被同一模块源文件中的任何实体访问，但是不能被模块外的实体访问。
	`internal` 是默认的访问级别

- `fileprivate` 限制实体只能在其定义的文件内部访问

- `private` 限制实体只能在其定义的作用域，以及同一文件内的 extension 访问
	`private`为最低访问级别(限制最多)

### 访问级别基本原则

`实体不能定义在具有更低访问级别（更严格）的实体中`

- 一个 public 的变量，其类型的访问级别不能是 internal，fileprivate 或是 private。因为无法保证变量的类型在使用变量的地方也具有访问权限。

- 函数的访问级别不能高于它的参数类型和返回类型的访问级别。因为这样就会出现函数可以在任何地方被访问，但是它的参数类型和返回类型却不可以的情况。

### 单target应用程序的访问级别

如果编写的代码只有一个`单独的Target`，则直接使用默认的`internal`即可，当然也可以使用更加严格的`private`或者`filePrivate`用来隐藏一些实现细节

### 框架或者库的访问级别

如果是开发一个给别人使用的`库`，则需要把对外的接口定义为`open`或者`public`, 不需要提供给外部调用的仍使用`internal`即可

### 自定义类指定访问级别

自定义类型指定访问级别后，会影响到内部的类型成员(属性、方法、构造器、下标)

* 指定private: 类型成员为private
* 指定filePrivate: 类型成员为 filePrivate
* 指定public/internal: 类型尘缘为 internal

```swift 

public class SomePublicClass {                  // 显式 public 类
    public var somePublicProperty = 0            // 显式 public 类成员
    var someInternalProperty = 0                 // 隐式 internal 类成员
    fileprivate func someFilePrivateMethod() {}  // 显式 fileprivate 类成员
    private func somePrivateMethod() {}          // 显式 private 类成员
}

class SomeInternalClass {                       // 隐式 internal 类
    var someInternalProperty = 0                 // 隐式 internal 类成员
    fileprivate func someFilePrivateMethod() {}  // 显式 fileprivate 类成员
    private func somePrivateMethod() {}          // 显式 private 类成员
}

fileprivate class SomeFilePrivateClass {        // 显式 fileprivate 类
    func someFilePrivateMethod() {}              // 隐式 fileprivate 类成员
    private func somePrivateMethod() {}          // 显式 private 类成员
}

private class SomePrivateClass {                // 显式 private 类
    func somePrivateMethod() {}                  // 隐式 private 类成员
}
```

### 元组类型

元组的访问级别将由元组中访问级别`最严格的类型来决定`。例如，如果你构建了一个包含两种不同类型的元组，其中一个类型为 `internal`，另一个类型为 `private`，那么这个元组的访问级别为 `private`。

### 函数类型

函数的访问级别根据访问级别最严格的参数类型或返回类型的访问级别来决定。

```swift
func someFunction() -> (PrivateClass, InternalClass) {
    // 此处是函数实现部分
    return (PrivateClass(), InternalClass())
}
```
![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7bx510vfgj31mm04ygmq.jpg)

该函数没有指定默认的访问级别，按道理应该是`internal`,但是由于其返回值是`private/internal`级别因此其返回值也是`private`级别，所以这个函数的访问级别应该是`private`，需要显式的标注上。

### 枚举类型

枚举成员的访问级别和该枚举类型相同，你不能为枚举成员单独指定不同的访问级别。

```swift
public enum CompassPoint {
    case north
    case south
    case east
    case west
}
```

`注意`: 如果枚举中的任何`原始值`或者`关联值`类型，都不能低于枚举类型的访问级别
![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7bxfbxfqrj31oo03y3z2.jpg)

### 子类

你可以继承同一模块中的所有有访问权限的类，也可以继承不同模块中被`open`修饰的类。一个子类的访问级别不得高于父类的访问级别。

不过我们可以通过某些方法，将父类限制使用的某些方法或者属性降低访问限制

例如

```swift
public class PublicClass {
    public var property: String
    init(property: String) {
        self.property = property
    }
    
    fileprivate func publicMethod() {
        
    }
}

class ClassA: PublicClass {
    override public func publicMethod() {
        print("classA publicMethod")
        super.publicMethod()
    }
}
```

`注意`: 这两个class定义在`同一个文件中`，否则`filePrivate`定义的`func`在其他文件中`无法访问`。

正常情况下，我们在其他类中是没办法调用PublicClass的publicMethod的，但是我们通过给PublicClass增加一个子类，同时在子类中重写 publicMethod 方法，并将其访问控制修改为public(小于等于 class的访问级别 public)。 这样的话我们就可以在外部使用对应方法了。

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7by1bf3wuj31mu072ab4.jpg)


### 常量、变量、属性、下标

常量、变量、属性不能拥有比它们的类型更高的访问级别

```swift

var fileP: FilePrivateClass?

private class PrivateClass {
    public var property: String?
}

```
fileP的类型为private，但是这个属性却被定义为一个internal的，这时候就会报错

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7by7nn5q3j31o602mwf5.jpg)

### Getter 和 Setter

常量、变量、属性、下标的 Getters 和 Setters 的访问级别和它们所属类型的访问级别相同。

Setter 的访问级别可以低于对应的 Getter 的访问级别，这样就可以控制变量、属性或下标的读写权限。在 var 或 subscript 关键字之前，你可以通过 fileprivate(set)，private(set) 或 internal(set) 为它们的写入权限指定更低的访问级别。

```swift 
struct TrackedString {
    private(set) var numberOfEdits = 0
    var value: String = "" {
        didSet {
            numberOfEdits += 1
        }
    }
}
```

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7bycdwtwuj31na07itau.jpg)


### 协议

协议中的每个方法或属性都必须具有和该协议相同的访问级别。

`注意`: 这里是相同的访问级别，而非小于等于，如果你定义了一个 public 访问级别的协议，那么该协议的所有实现也会是 public 访问级别。这一点不同于其他类型，例如，类型是 public 访问级别时，其成员的访问级别却只是 internal。

#### 协议继承

如果定义了一个继承自其他协议的新协议，那么新协议拥有的访问级别最高也只能和被继承协议的访问级别相同。例如，你不能将继承自 internal 协议的新协议访问级别指定为 public 协议。

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7byhs570aj31pi08kq49.jpg)

#### 协议遵循

一个类型可以遵循比它级别更低的协议,协议实现的访问级别必须大于等于协议里那个方法的访问级别

正确:

```swift
internal protocol PublicProtocol {
    func test() // internal
}

public class PublishClass: PublicProtocol {
    func test() {
        // 默认internal
    }
}
```

错误：

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7bzeg5n9dj31og09o407.jpg)


### Extension

Extension 可以在访问级别允许的情况下对类、结构体、枚举进行扩展。Extension 的新增成员具有和原始类型成员一致的访问级别。

但是需要注意，extension的访问级别只允许小于类型定义时的访问级别

例如: 一直ViewController为internal 

正确:

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7croirpgxj31p604gaac.jpg)

错误:

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7crp10kb7j31p604kgm2.jpg)

你也可以通过修饰语重新指定 extension 的默认访问级别，但是如果你使用 extension 来遵循协议的话，就不能显式地声明 extension 的访问级别。extension 每个 protocol 要求的实现都默认使用 protocol 的访问级别。

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h7crqlex7wj31p006awfj.jpg)


