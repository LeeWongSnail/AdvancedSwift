##  didset中调用set方法 会导致循环吗？

我们先来看下这段代码:

```swift
class ModalStudent {
    var name: String? {
        didSet {
            print(oldValue)
            print(name)
            name = "Wong"
        }
    }
}

private func stuTest() {
    let student = ModalStudent()
    student.name = "Lee"
    print(student.name)
}
```

请问上述代码是否有什么问题？最终的输出结果是什么呢？name的最终值是什么呢？

先说结果，经过验证，代码运行没有问题，最终输出结果为:

```
nil
Optional("Lee")
Optional("Wong")
```

根据输出我们可以得到下述结论:

- 在didSet方法中调用属性的设置方法不会导致死循环
- 外部设置name属性时name已被赋值，oldValue表示的是该属性之前的值
- 在didSet中设置属性值是成功的

