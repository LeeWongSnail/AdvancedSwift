## 'self' refers to the method 'x', which may be unexpected

今天线上突然反馈一个问题，用户在iPhone12(iOS15.4)，在使用我们App是点击某个按钮无响应。

当我打开对应文件找到这个按钮对应的代码位置是发现

```swift
private let addButton: UIButton = {
    let button = UIButton(type: .custom)
    button.setImage(UIImage(named: "icon_add_white"), for: .normal)
    button.setTitle(NSLocalizedString("Add Signature", comment: "Add Signature"), for: .normal)
    button.setTitleColor(UIColor.white, for: .normal)
    button.titleLabel?.font = UIFont.systemFont(ofSize: 12, weight: .regular)
    button.addTarget(self, action: #selector(addButtonTapped(_:)), for: .touchUpInside)
    return button
}()
```

我发现编译器给我提示了一个警告(目前使用的是Xcode14.0)：

![xcodewarningletproperty](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/xcodewarningletproperty.png)

很奇怪的警告，但是当我看到这种写法时顿时觉得更加奇怪，为什么要这么定义一个UI控件呢？

### 警告原因

警告提示我们`'self' refers to the method 'SignatureFooterTrayView.self', which may be unexpected`,意思很明确，这里的self可能指的是`SignatureFooterTrayView.self`,这可能不是我们所期望的。

什么鬼，这肯定不是我们期望的，我们期望的是SignatureFooterTrayView的实例，而不是他的类型(实例方法而非类方法)。

我们尝试使用Xcode的提示自动帮我们修改:

```swift
private let addButton: UIButton = {
	let button = UIButton(type: .custom)
	button.setImage(UIImage(named: "icon_add_white"), for: .normal)
	button.setTitle(NSLocalizedString("Add Signature", comment: "Add Signature"), for: .normal)
	button.setTitleColor(UIColor.white, for: .normal)
	button.titleLabel?.font = UIFont.systemFont(ofSize: 12, weight: .regular)
	button.addTarget(SignatureFooterTrayView.self, action: #selector(addButtonTapped(_:)), for: .touchUpInside)
    return button
}()
```
果然系统给我们的建议与错误的描述一致

```swift
	button.addTarget(SignatureFooterTrayView.self, action: #selector(addButtonTapped(_:)), for: .touchUpInside)
```
这表示在按钮点击时会调用`SignatureFooterTrayView`的`addButtonTapped`这个类方法。

这很明显不是我们想要的，而解决这个警告也很简单，我们只需要把这个属性改成懒加载即可，而且对于UI控件我们也习惯使用懒加载

```swift
private lazy var addButton: UIButton = {
   let button = UIButton(type: .custom)
   button.setImage(UIImage(named: "icon_add_white"), for: .normal)
   button.setTitle(NSLocalizedString("Add Signature", comment: "Add Signature"), for: .normal)
   button.setTitleColor(UIColor.white, for: .normal)
   button.titleLabel?.font = UIFont.systemFont(ofSize: 12, weight: .regular)
   button.addTarget(self, action: #selector(addButtonTapped(_:)), for: .touchUpInside)
   return button
}()
```
而且经过验证，我们这种改法也是有效的，按钮点击事件响应正确.

### 刨根问底

那么为什么刚才的那种写法会导致这个问题呢？

我们来分析下上面代码的执行过程:

- addButton是一个存储属性，且是一个常量，它会立即通过匿名闭包完成初始化
- Swift的初始化规则是:存储属性必须在拥有他的类型初始化之前初始化
- 因此，在addButton被初始化时(闭包执行时)，self还是nil
- 方法`addTarget(_:, action:, for:)`接受一个`Any?`的值，因此这里传nil也是正常的
- 因此在执行时self是nil，编译器认为这里是nil是有问题的所以提示我们可能我们期望这里是非nil的`SignatureFooterTrayView.self `

我们来确认下上面的想法:

```swift
override init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?) {
    super.init(nibName: nibNameOrNil, bundle: nibBundleOrNil)
}


private let addButton: UIButton = {
    let button = UIButton(type: .custom)
    button.setImage(UIImage(named: "icon_add_white"), for: .normal)
    button.setTitle(NSLocalizedString("Add Signature", comment: "Add Signature"), for: .normal)
    button.setTitleColor(UIColor.white, for: .normal)
    button.titleLabel?.font = UIFont.systemFont(ofSize: 12, weight: .regular)
    button.addTarget(self, action: #selector(addButtonTapped(_:)), for: .touchUpInside)
    return button
}()
```
首先先来确认下哪个方法先执行:

![privateletfirst](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/privateletfirst.png)

那么我们来通过断点确认下这时候self是什么？

我们在addButton的初始化block中添加下面两句:

```swift
print(self)
print(type(of: self))
```
结果输出:

```
(Function)
(ViewController) -> () -> ViewController
```
self实际是一个Function,这个function是闭包结果是返回一个viewController

我的这个方法命名是要返回一个UIButton为何这里是返回一个viewController???

这个问题目前还没有了解具体原因到底是什么？


如果你知道请告知我一下 please 😊

### 参考文献

[What type is self in a Swift self-executing anonymous closure used to initialize a stored property?](https://www.jessesquires.com/blog/2020/12/22/swift-self-executing-anonymous-closures/)

[Xcode 13.3 warning: "'self' refers to the method '{object}.self', which may be unexpected](https://stackoverflow.com/questions/71560311/xcode-13-3-warning-self-refers-to-the-method-object-self-which-may-be-u)

['self' refers to the method 'EquationController.self', which may be unexpected](https://stackoverflow.com/questions/72018007/self-refers-to-the-method-equationcontroller-self-which-may-be-unexpected)

[[SR-4865] target self should be illegal in property constructor](https://github.com/apple/swift/issues/47442)

[[SR-4559] Method called 'self' can be confused with regular 'self' #47136](https://github.com/apple/swift/issues/47136)

