## unowned/weak?

### 背景

在日常的开发中，我们经常会遇到内存问题，遇到内存后就不可避免的会遇到`weak`和`unowned`这两个关键词。在之前的开发中，本着不出错的原则，我基本上都在使用`weak`，对于`unowned`基本上没有使用(`crash风险`)。但是之前并没有理解这两个的深层次区别到底是什么，`仅仅是因为weak会在执行的对象被释放后指针会置nil吗`？ 

这篇文章我们来详细的了解下



### 循环引用

在ARC之前的MRC，我们就不多讲了毕竟距离现在也是有些遥远了，目前应该是没有要维护的MRC项目了。Swift更是一出生就使用ARC，如果对ARC不了解只需要知道一句话:`每一个引用类型实例都有一个引用计数与之关联，这个引用计数用来记录这个对象实例正在被变量或常量引用的总次数。当引用计数变为 0 时，实例将会被析构，实例占有的内存和资源都将变得重新可用。`

但是在我们平时的开发中可能会遇到这种问题:

```swift
class BabySister {
    var name: String
    var baby: Baby?
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("BabySister \(name) deinit")
    }
}

class Baby {
    var name: String
    var babySister: BabySister?
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("Baby \(name) deinit")
    }
}
```
假设我们有上面两个类`BabySister`和`Baby`，我们知道每个`baby`都应该有一个`babySister`，而每个babySister也只照顾一个baby。

那么他们之间的关系可以描述为

```swift
var babySister: BabySister? = BabySister(name: "Red")
var baby: Baby? = Baby(name: "Lee")

babySister?.baby = baby
baby?.babySister = babySister
```

但是当有一个baby足够大了，babySister也被解雇之后，我们期望他们之间的关系到此结束

```swift
// baby sister is fired and baby old enough
babySister = nil
baby = nil
```
但实际并没有，因为: Baby和BabySister之间还在互相持有。这就导致你还要为这段关系支付`工资`。

![babyandbabysister](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/babyandbabysister.png)

### 捕获列表

我们在OC的开发中经常遇到在Block中导致循环引用的问题，那是因为在block中会捕获block外部的变量并进行强引用。在Swift中引入了一个新的概念叫做`捕获列表(capture list)`,使用捕获列表，可以在函数的头部`定义和指定那些需要用在内部的外部变量，并且指定引用类型(weak/unowned)`。

我们来看下下面这个例子:

```swift
var i1 = 1, i2 = 1

var fStrong = {
    i1 += 1
    i2 += 2
}

fStrong()
print(i1,i2) //Prints 2 and 3
```
当我们调用`fStrong`时，会直接修改外部变量的值

假如我们把上面的代码修改下:

```swift
var fCopy = { [i1] in
    print(i1,i2)
}

fStrong()
print(i1,i2) //打印结果是 2 和 3  

fCopy()  //打印结果是 1 和 3
```

我们在`fStrong`执行前面添加了一个`fCopy`,同时使用捕获列表指定`i1`(这里并没有指定引用类型)

这里闭包内部会`创建一个新的可用常量`。没有指定常量的修饰符，`闭包会简单的拷贝原始值到新的变量中`，对于值类型和引用类型都是一样的。

在上面的例子中，在`fStrong`调用前创建了`fCopy`，在`fCopy`被定义的时候，私有的常量`已经被创建`了，因此在`fCopy`被调用的时候输出结果i1的值仍为1。

由于闭包进行的是拷贝操作，因此:

- 对于值类型，仅仅是值的复制，这个值不在受外部的影响
- 对于引用类型，会使其引用计数(retain count) +1

假如我们在捕获列表中指定引用类型(weak/unowned),这个拷贝的常量(引用类型)会被初始化为一个弱引用，指向原始值，我们就可以通过这种方式避免因为闭包导致引用计数器增加。

如下例:

```swift
class aClass{
    var value = 1
}

var c1 = aClass()
var c2 = aClass()

var fSpec = { [unowned c1, weak c2] in
    c1.value += 1
    if let c2 = c2 {
        c2.value += 1
    }
}

fSpec()
print(c1.value,c2.value) //Prints 2 and 2
```

捕获列表中的两个变量的类型不同，决定了他们在闭包中的使用方式也不同

- unowned引用的使用场景是: 原始值实例永远不可能为nil，闭包可以直接使用他，并且直接定义为显示解包可选值，但是当实例被释放后，在闭包中使用这个捕获值可能会导致崩溃。

- weak引用的使用场景: 原始值可能为nil,因此在使用是必须进行判空等逻辑 


### 不是所有的block都需要weak/unowned

在某些场景下，例如

```swift
class Blog {
    let name: String
    let url: URL
    weak var owner: Blogger?

    init(name: String, url: URL) { self.name = name; self.url = url }

    deinit {
        print("Blog \(name) is being deinitialized")
    }
}

class Blogger {
    let name: String
    var blog: Blog?

    init(name: String) { self.name = name }

    deinit {
        print("Blogger \(name) is being deinitialized")
    }
}
struct Post {
    let title: String
    var isPublished: Bool = false

    init(title: String) { self.title = title }
}

class Blog {
    let name: String
    let url: URL
    weak var owner: Blogger?

    var publishedPosts: [Post] = []

    init(name: String, url: URL) { self.name = name; self.url = url }

    deinit {
        print("Blog \(name) is being deinitialized")
    }

    func publish(post: Post) {
        /// Faking a network request with this delay:
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            self.publishedPosts.append(post)
            print("Published post count is now: \(self.publishedPosts.count)")
        }
    }
}

var blog: Blog? = Blog(name: "SwiftLee", url: URL(string: "www.avanderlee.com")!)
var blogger: Blogger? = Blogger(name: "Antoine van der Lee")

blog!.owner = blogger
blogger!.blog = blog

blog!.publish(post: Post(title: "Explaining weak and unowned self"))
blog = nil
blogger = nil

```
这段代码的输出是:

```
// Blogger Antoine van der Lee is being deinitialized
// Published post count is now: 1
// Blog SwiftLee is being deinitialized
```
这种情况下，虽然在publish返回之前 blog已经被置为nil了，但是因为self，我们是的这次pushlish结果得以保存。

如果我们修改为

```swift
func publish(post: Post) {
    /// Faking a network request with this delay:
    DispatchQueue.main.asyncAfter(deadline: .now() + 1) { [weak self] in
        self?.publishedPosts.append(post)
        print("Published post count is now: \(self?.publishedPosts.count)")
    }
}
```

控制台输出为

```swift
// Blogger Antoine van der Lee is being deinitialized
// Blog SwiftLee is being deinitialized
// Published post count is now: nil
```
使用[weak self]后，一旦blog被置为nil则延时过后因为block没有持有self,这时候self已经为nil了，所以我们无法将对应数据保存


### weak还是unowned?

通过上面的了解我们可以很清楚的了解这两者之间的区别，那么使用场景也很清晰: 由原始值的生命周期来决定

![weakorunowned](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/weakorunowned.png)


这就对应着两种场景：

- 闭包和捕获对象的生命周期相同，所以对象可以被访问，也就意味着闭包也可以被访问。外部对象和闭包有相同的生命周期。在这种情况下，你应该把引用定义为 unowned。

- 闭包的生命周期和捕获对象的生命周期相互独立，当对象不能再使用时，闭包依然能够被引用。这种情况下，你应该把引用定义为 weak，并且在使用它之前验证一下它是否为 nil（请不要对它进行强制解包).

其实官方解释也很清楚:

```
Use a weak reference whenever it is valid for that reference to become nil at some point during its lifetime. Conversely, use an unowned reference when you know that the reference will never be nil once it has been set during initialization.
```


### 结论

那么，当无法确认两个对象之间生命周期的关系时，是否不应该去冒险选择一个无效 unowned 引用？而是保守选择 weak 引用是一个更好的选择？

答案是否定的，因为声明周期使我们必须要了解的一件事情，而且这两种类型的区别也不止于此。

弱引用的常见方式是，没生成一个新的引用，都会报这个弱引用和他指向的对象的信息保存到一个表中(SideTable),当一个对象没有被强引用时(retainCount=0)这个对象就会被释放，但是在释放之前运行时会把所有相关的弱引用置为nil(weak/unowned).

这种实现实际是由内存开销的，考虑到需要额外实现的数据结构，需要保证在并发情况下这个操作表(SideTable)所有操作的正确性。一旦开始析构某个对象，在任何环境下都不允许在访问弱引用锁指向的原始对象了。

Swift中的每个对象都保存了两个引用技术: 
- ①强引用计数  用来决定这个对象什么时机可以被释放(开始析构)
- ②弱引用计数 用来计算创建了多少个指向这个对象的弱引用(weak/unowned),当计数器为0时对象被西沟

`重点`: 只有当所有的`unowned`引用被释放后，这个对象才会被真正的析构，析构开始时对象会保持`未解析可访问`状态，析构发生后，对象内容才会被回收。

在编译时，当Optimization Level被设置为 -OFast时, unowned引用不会验证引用对象的有效性，unowned 引用的行为就会像 Objective-C 中的 __unsafe_unretained 一样。如果引用对象无效，unowned 引用将会指向已经释放垃圾内存（这种实现称之 unowned(unsafe)）。

![unownedoptimizationlevel](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/unownedoptimizationlevel.png)

当一个unowned引用被释放后，如果这回收没有其他强引用或者unowned引用指向这个对象，那么这个对象最终将会被析构，而非强引用计数器为零的情况下被析构。

这里针对weak和unowned的具体实现我们不做过多讲解，后续会单独拿出来一片文章讲解。


### 参考文献

[内存管理，WEAK 和 UNOWNED](https://swifter.tips/retain-cycle/)

[Unowned 还是 Weak？生命周期和性能对比](https://swift.gg/2017/05/16/unowned-or-weak-lifetime-and-performance/)

[Weak self and unowned self explained in Swift](https://www.avanderlee.com/swift/weak-self/)

