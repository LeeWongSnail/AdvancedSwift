# swift中单例的正确用法

单例顾名思义就是要在整个生命周期中只保存一份。同时也以为这个这个对象在APP的声明周期中会一直存在且不会被销毁。

## 何为单例

对于单例，必须满足下面这几个条件:

- 唯一性
- 创建方法不对外部开放
- 线程安全

## OC中的单例

我们先来看下在OC中单例的实现:

```objc
+ (instancetype)sharedManager
{
	static ESCScanSessionManager *sharedManager;
	static dispatch_once_t onceToken;
	dispatch_once(&onceToken, ^{
		sharedManager = [self new];
	});
	return sharedManager;
}
```

现在我们通过单例的三个条件来重新审视一下这种写法。

- 唯一性 我们通过dispatch_once来保证方法只被执行一次
- 上面的方法并没有满足创建方法不对外部开放 外部仍可以通过 [ESCScanSessionManager new]来创建一个新的非单例对象
- dispatch_once同样也是我们可以保证只被执行一次，也就是说即使多线程同时访问也不会多次创建因此也就解决了线程安全的问题

## swift中的单例

在swift早期的时候有很多方法来书写swift的单例，大家可以自行百度，我们里就不做赘述。
我们之看下最近常用的两种:

### 全局变量写法

```swift
private let sharedKraken = TheOneAndOnlyKraken()
class TheOneAndOnlyKraken {
	class var sharedInstance: TheOneAndOnlyKraken {
		return sharedInstance
	}
}
```
同样我们用单例的三个条件来判断他是否满足单例的要求:

* 1、唯一性
我们通过断点发现，定义的全局private let 是懒加载的，第一次使用的时候才会被调用，同时

![shareinstance_once]()

实际上全局的常量也是通过dispatch_once来实现的所以可以保证智慧调用一次

* 2、不对外开放创建方法

实际上上述写法并没有显示的写出init方法是否私有， 因此我们还是可以通过默认初始化器产生一个新的类。

* 3、线程安全
dispatch_once可以保证初始化操作的原子性，因此也是线程安全的

### 完美的方式

```swift
class TheOneAndOnlyKraken {
	static let sharedInstance = TheOneAndOnlyKraken()
	private init() {} // 这就阻止其他对象使用这个类的默认的'()'初始化方法
}
```
还是老规矩 看他是否满足那三个条件：

#### 唯一性

我们还是通过断点的方式

![shareInstance_perfect]()

定义在类内部的let也是通过dispatch_once来保证唯一性的，因此这个条件是肯定满足的


#### 不对外开放初始化

我们看到了在类中的init方法被定义为私有，因此当我们想在外部初始化这个类时：

![TheOneAndOnlyKraken]()

编译器提示我们`'TheOneAndOnlyKraken' initializer is inaccessible due to 'private' protection level`,因此我们无法在外部新建该类型的实例

这里如果是单例继承自NSObject则需要

```swift
class TheOneAndOnlyKraken: NSObject {
    static let sharedInstance = TheOneAndOnlyKraken()
    let name: String = "111"
    private override init() {} // 这就阻止其他对象使用这个类的默认的'()'初始化方法
}
```
这样我们外部也是无法进行初始化的。

#### 线程安全

这一点也不用质疑 dispatch_once已经保证了原子操作因此不存在多线程可能导致多次创建的可能性。


### 可以释放的单例

我们在最开始描述单例的时候，我们用了一个词`永远不会被释放`。那么如果创建一个可能被释放的单例呢？

因为我们发现有些场景的单例，持有了太多东西，从而导致很多对象无法被释放，所以在某些场景下我们希望这个单例可以被释放。

```swift
class TheOneAndOnlyKraken {
    private static var _sharedInstance: TheOneAndOnlyKraken?
    class func getSharedInstance() -> TheOneAndOnlyKraken {
        guard let instance = _sharedInstance else {
            _sharedInstance = TheOneAndOnlyKraken()
            return _sharedInstance!
        }
        return instance
    }

    private init() {} //私有化init方法
    
    deinit {
        print("\(self) is being deinit")
    }
    
    //销毁单例对象
    class func destroy() {
        _sharedInstance = nil
    }
}
```

我们尝试调用:

```swift
var instance: TheOneAndOnlyKraken? = TheOneAndOnlyKraken.getSharedInstance()
print(instance)
TheOneAndOnlyKraken.destroy()
```
输出结果为:

```
Optional(SwiftDemo.AppManager)
SwiftDemo.AppManager is being deinit
```

这样我们的单例就可以释放了

### 总结

虽然单例的写法有很多中，但是只要我们知道单例必须满足的三个条件，我们就可以一一的去分析，然后判断是否是一个合理的单例写法。



