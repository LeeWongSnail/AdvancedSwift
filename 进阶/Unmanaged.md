## Unmanaged

Unmanaged 表示对不清晰的内存管理对象的封装.

先来看下这个类在Swift源码中的实现:

```swift
@frozen
public struct Unmanaged<Instance: AnyObject> {
	public func takeUnretainedValue() -> Instance {
    	return _value
	}
  	public func takeRetainedValue() -> Instance {
    	let result = _value
    	release()
    	return result
  	}
	public func retain() -> Unmanaged {
    	Builtin.retain(_value)
    	return self
 	}
 	public func release() {
    	Builtin.release(_value)
  	}
  	public func autorelease() -> Unmanaged {
    	Builtin.autorelease(_value)
    	return self
  	}
```

很奇怪，这个类竟然还有retain和release这种方法，这类方法早在使用ARC后就已经被弃用了，为何这里还会存在呢？

 
### 背景

在OC中一开始的MRC时代，我们都是还需要手动进行内存管理，每一个retain都要对应一个release，这样才能避免内存泄露和僵尸对象。自从ARC引入后，我们就不需要手动的进行内存管理，下图就是ARC示意图:

![unmanaged_arc](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/unmanaged_arc.png)

但是，即使使用了ARC也并不意味着所有的类型都不需要手动管理内存了，我们来看下下面这段代码:

```objc
- (NSDictionary *)imagePropertiesForImageURL:(NSURL *)imageURL
{
    NSDictionary *imageProperties = nil;
    
    CGImageSourceRef imageSource = CGImageSourceCreateWithURL((__bridge CFURLRef) imageURL, NULL);
    
    if (imageSource)
    {
        if (CGImageSourceGetCount(imageSource))
            imageProperties = (__bridge_transfer NSDictionary *) CGImageSourceCopyPropertiesAtIndex(imageSource, 0, NULL);
        
        CFRelease(imageSource);
    }
    
    return imageProperties;
}
```

```objc
NSURL *fileURL = [NSURL URLWithString:@"SomeURL"];
SystemSoundID theSoundID;
//OSStatus AudioServicesCreateSystemSoundID(CFURLRef inFileURL,
//                             SystemSoundID *outSystemSoundID);
OSStatus error = AudioServicesCreateSystemSoundID(
        (__bridge CFURLRef)fileURL,
        &theSoundID);
```
虽然是在ARC中，但是我们依然需要调用CFRelease来释放对象(引用计数器减一)。这是因为:

`在现在这个后 ARC 的世界里，所有的 Objective-C 和 从 Objective-C 方法返回的 Core Foundation 类型的内存都被自动管理，只剩下由 C 函数返回的 Core Foundation 类型还没有收编。对于后者而言，对象所有权的管理仍然停留在调用 CFRetain() 和 CFRelease()、或通过某个 __bridge 函数桥接到 Objective-C 对象的方式的层面上。`

#### Create&Get规则

为了帮助大家理解`C`函数返回对象是否被调用者持有，苹果使用了`Create`规则和`Get`规则命名法：

- `Create` 规则 的意思是，如果一个函数的名字含有 `Create` 或 `Copy` ，函数的返回值被函数的调用者持有。也就是说，调用 `Create` 或 `Copy` 函数的对象应该对返回对象调用 `CFRelease` 进行释放。

- `Get` 规则 则不像 Create 规则一样能从命名规则看出规律。或许可以描述成函数名不含有 `Create` 或`Copy` 的函数？这种函数遵守`Get`规则，返回对象的持有者不会发生变化。如果想持久化一个返回对象，大多数时候就是你自己手动 retain 它。


`Swift 仅支持 ARC`，所以也没有地方调用`CFRelease`或 `__bridge_retained`。那么`Swift` 是如何让这种 `“在上下文中内存管理”` 的哲学融入自己的内存安全体系呢？

事情分两种情况。

- 注明 的 API，Swift 能够在上下文中严格遵循注释描述对 CoreFoundation API 进行内存管理，并以同样内存安全的方式桥接到 Objective-C 或 Swift 类型上。
- 没有明确注明的 API，Swift 则会通过 Unmanaged 类型把工作交给开发者。


### 管理 Unmanaged

虽然大多数的 CoreFoundation API 都有注明是否可自动管理，但一些重要的部分还没有得到充分重视。

```swift
@available(iOS 10.0, *)
public func SecKeyCopyExternalRepresentation(_ key: SecKey, _ error: UnsafeMutablePointer<Unmanaged<CFError>?>?) -> CFData?
```

一个 Unmanaged<T> 实例封装有一个 CoreFoundation 类型 T，它在相应范围内持有对该 T 对象的引用。从一个 Unmanaged 实例中获取一个 Swift 值的方法有两种：

- `takeRetainedValue()`：返回该实例中 Swift 管理的引用，并在调用的同时`减少一次引用次数`，所以可以按照 `Create` 规则来对待其返回值。
- `takeUnretainedValue()`：返回该实例中 Swift 管理的引用而 `不减少` 引用次数，所以可以按照 Get 规则来对待其返回值。

在实践中最好不要直接操作 Unmanaged 实例，而是用这两个 take 开头的方法从返回值中拿到绑定的对象。

上面提到在`Objective-C`时代`ARC`不能处理的一个问题是`CF`类型的创建和释放。虽然不能自动化，但是遵循命名规则来处理的话还是比较简单的：

`对于 `CF` 系的 API，如果 API 的名字中含有` Create`，`Copy` 或者 `Retain` 的话，在使用完成后，我们需要调用 `CFRelease` 来进行释放。

不过 `Swift` 中这条规则已成明日黄花。既然我们有了明确的规则，那为什么还要一次一次不厌其烦地手动去写 `Release `呢？基于这种想法，Swift 中我们不再需要显式地去释放带有这些关键字的内容了 (事实上，`含有 CFRelease 的代码甚至无法通过编译`)。也就是说，`CF 现在也在 ARC 的管辖范围之内了`。其实背后的机理一点都不复杂，只不过在合适的地方加上了像 `CF_RETURNS_RETAINED` 和 `CF_RETURNS_NOT_RETAINED` 这样的标注。

但是有一点例外，那就是对于非系统的 `CF API` (比如你自己写的或者是第三方的)，因为并没有强制机制要求它们一定遵照 Cocoa 的命名规范，所以贸然进行自动内存管理是不可行的。如果你没有明确地使用上面的标注来指明内存管理的方式的话，将这些返回 CF 对象的 API 导入 Swift 时，它们的类型会被对对应为 `Unmanaged<T>`。

```swift
@discardableResult
open func startListening() -> Bool {
    var context = SCNetworkReachabilityContext(version: 0, info: nil, retain: nil, release: nil, copyDescription: nil)
    context.info = Unmanaged.passUnretained(self).toOpaque()

    let callbackEnabled = SCNetworkReachabilitySetCallback(
        reachability,
        { (_, flags, info) in
            let reachability = Unmanaged<NetworkReachabilityManager>.fromOpaque(info!).takeUnretainedValue()
            reachability.notifyListener(flags)
        },
        &context
    )

    let queueEnabled = SCNetworkReachabilitySetDispatchQueue(reachability, listenerQueue)

    listenerQueue.async {
        self.previousFlags = SCNetworkReachabilityFlags(rawValue: 1 << 30)

        guard let flags = self.flags else { return }

        self.notifyListener(flags)
    }

    return callbackEnabled && queueEnabled
}
```

```swift
@available(iOS 10.0, watchOS 3.0, tvOS 10.0, *)
static func generateRSAKeyPair(sizeInBits size: Int, applyUnitTestWorkaround: Bool = false) throws -> (privateKey: PrivateKey, publicKey: PublicKey) {
      
   guard let tagData = UUID().uuidString.data(using: .utf8) else {
       throw SwiftyRSAError.stringToDataConversionFailed
   }
   
   // @hack Don't store permanently when running unit tests, otherwise we'll get a key creation error (NSOSStatusErrorDomain -50)
    // @see http://www.openradar.me/36809637
    // @see https://stackoverflow.com/q/48414685/646960
    let isPermanent = applyUnitTestWorkaround ? false : true
    
    let attributes: [CFString: Any] = [
        kSecAttrKeyType: kSecAttrKeyTypeRSA,
        kSecAttrKeySizeInBits: size,
        kSecPrivateKeyAttrs: [
            kSecAttrIsPermanent: isPermanent,
            kSecAttrApplicationTag: tagData
        ]
    ]
    
    var error: Unmanaged<CFError>?
    guard let privKey = SecKeyCreateRandomKey(attributes as CFDictionary, &error),
        let pubKey = SecKeyCopyPublicKey(privKey) else {
        throw SwiftyRSAError.keyGenerationFailed(error: error?.takeRetainedValue())
    }
    let privateKey = try PrivateKey(reference: privKey)
    let publicKey = try PublicKey(reference: pubKey)
    
    return (privateKey: privateKey, publicKey: publicKey)
}
```

*上述代码是我们自Alamofire的NetworkReachabilityManager以及SwiftyRSA中SwiftyRSA文件摘抄的一段代码，这里用到了Unmanaged。*

这意味着在使用时我们需要手动进行内存管理，
一般来说会使用得到的 `Unmanaged` 对象的 `takeUnretainedValue` 或者 `takeRetainedValue `从中取出需要的 CF 对象，并同时处理引用计数。

`takeUnretainedValue` 将保持原来的引用计数不变，在你明白你没有义务去释放原来的内存时，应该使用这个方法。而如果你需要释放得到的 CF 的对象的内存时，应该使用 `takeRetainedValue` 来让引用计数加一，

然后在使用完后对原来的 `Unmanaged` 进行手动释放。为了能手动操作 `Unmanaged` 的引用计数，`Unmanaged` 中还提供了 `retain`，`release` 和 `autorelease` 这样的 "老朋友" 供我们使用。


### 最好的解决问题的方法是避免遇到问题

现在我们已经知道如何对付`Unmanaged` 了，现在我们还是看看如何`避免`碰到这种情况吧。如果 `Unmanaged` 引用是从你自己写的 C 函数返回的，那么你最好还是注明一下。这种注释能够帮助编译器理解如何自动管理你所返回对象的内存：
就不要用 Unmanaged<CFString> 了，直接返回一个在 Swift 中类型安全以及内存管理完善的 CFString 类型。

举例说明，我们有一个函数能将两个`CFString`对象拼装成一个字符串，并且要告诉`Swift`这个返回字符串的内存是被如何管理的。根据上面提到的命名规则，我们的函数应该叫做 `CreateJoinedString` —— 这个名字表达的意思是调用者将持有返回值。

```swift
CFStringRef CreateJoinedString(CFStringRef string1, CFStringRef string2);
```

既然这样，在函数实现中我们用 `CFStringCreateMutableCopy` 创建的 `resultString` 返回时没有与其创建函数平衡的 `CFRelease`：

```swift
CFStringRef CreateJoinedString(CFStringRef string1, CFStringRef string2) {
    CFMutableStringRef resultString = CFStringCreateMutableCopy(NULL, 0, string1);
    CFStringAppend(resultString, string2);
    return resultString;
}
```

当我们在Swift中引用上面的方法时:

```swift
// imported declaration:
func CreateJoinedString(string1: CFString!, string2: CFString!) -> Unmanaged<CFString>!
```

调用时:

```swift
// to call:
let joinedString = CreateJoinedString("First", "Second").takeRetainedValue() as String
```

既然我们的函数遵循了`Create`规则进行命名，那么就可以打开编译器的隐式桥接来消除`Unmanaged`歧义。`Core Foundation` 提供了两个宏：`CF_IMPLICIT_BRIDGING_ENABLED` 和 `CF_IMPLICIT_BRIDGING_DISABLED `—— 用来打开和关闭 Clang 的 `arc_cf_code_audited` 变量：

```
CF_IMPLICIT_BRIDGING_ENABLED            // get rid of Unmanaged
#pragma clang assume_nonnull begin      // also get rid of !s

CFStringRef CreateJoinedString(CFStringRef string1, CFStringRef string2);

#pragma clang assume_nonnull end
CF_IMPLICIT_BRIDGING_DISABLED
```

现在Swift已经能够控制这个函数返回值的内存管理了，我们的代码里也不需要Unmanaged了:

```swift
// imported declaration:
func CreateJoinedString(string1: CFString, string2: CFString) -> CFString

// to call:
let joinedString = CreateJoinedString("First", "Second") as String
```

如果你的函数 没有使用`Create/Get` 规则来命名，那么明显地，你把这些函数用这个法则重新命名一次。

当然在真实情况下这种修改可能并不容易，但是拥有明确性一致性返回的 API 的好处不仅仅是避免 `Unmanaged`。

如果不能够重命名，也有另外两种注明方式可以使用：

- 将持有者转移到调用者的函数应该使用 CF_RETURNS_RETAINED，
- 则使用 CF_RETURNS_NOT_RETAINED。

比如说，这个命名糟糕的 MakeJoinedString 就是用了手动注明的方式来表明其性质：

```swift
CF_RETURNS_RETAINED
__nonnull CFStringRef MakeJoinedString(__nonnull CFStringRef string1,
                                       __nonnull CFStringRef string2);
```


### 参考文献

[TOLL-FREE BRIDGING 和 UNMANAGED](https://swifter.tips/)

[Unmanaged](https://nshipster.cn/unmanaged/)

[Unmanaged 英文版](https://nshipster.com/unmanaged/)














