## Swift类型推断

Swift是一种静态类型语言，这就意味着我们声明的每个属性和变量他的类型都是需要在编译时就指定好的。但是在实际的开发中，我们经常不指定类型，但是编译器没有提示任务错误，这又是为什么呢？

这实际上就是类型推断的功能。

类型推断，顾名思义就是即使不显式的去标注类型，编译器可以根据上下文去推断变量的类型。

类型推断可以帮助开发者编写更加干净、简洁的代码，而不损害类型安全。

- 类型推断允许你省略掉一些次要的细节(Type inference allows you to omit incidental details)

![](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/classNameType.png)

### 类型推断的过程

我们已WWDC20中提供的例子来了解类型推断的实现过程

我们先来看下下面这个示例

```swift
struct SmoothieList: View {
    var smoothies: [Smoothie]
    @State var searchPhrase = ""
    
    var body: some View {
        FilteredList(
            smoothies,
            filterBy: \.title,
            isIncluded: {title in title.hasSubstring(searchPhrase)}
        ) { smoothie in
            SmoothieRowView(smoothie: smoothie)
        }
    }
}
```
这是一个SwiftUI的部分代码，主要实现的功能是 我要在一个`smoothies`列表中根据搜索关键词(`searchPhrase`)来筛选符合关键词的结果。
当前的页面`body`是一个`list`,每一行是一个`SmoothieRowView`。

我要做的就是对数据源进行筛选，然后通过筛选后的数据源重新生成一个列表，筛选条件是：`数据源中title字段包含搜索关键词的数据`

那么我们看下FilteredList应该如何实现

```swift
public struct FilteredList<Element, FilterKey, RowContent> {
    public init(_ data: [Element],
                filterBy key: KeyPath<Element, FilterKey>),
    @ViewBuilder rowContent: @escaping (Element) -> RowContent) {}
}
```

这里我们定义了三个占位类型

- Element 表示数据类型 即数据源中每个数据的类型 我们的第一个参数就是这个类型的数组
- FilterKey 要筛选字段 也是作为后面block参数的入参 表示数据源类型中的哪一个属性
- RowContent 则是要作为返回值

然我们我们看下根据第一段代码给出的

#### 1
```swift
FilteredList<Element, FilterKey, RowContent> (
    smoothies,
    filterBy: \.title,
    isIncluded: {title in title.hasSubstring(searchPhrase)}
) { smoothie in
        SmoothieRowView(smoothie: smoothie)
}
```

#### 2

```swift
FilteredList<Element, FilterKey, RowContent>(
    smoothies as [Element],
    filterBy: \.title,
    isIncluded: {title in title.hasSubstring(searchPhrase)}
) { smoothie in
        SmoothieRowView(smoothie: smoothie)
}
```

#### 3

```swift
FilteredList<Element, FilterKey, RowContent>(
    smoothies as [Element],
    filterBy: \Element.title as KeyPath<Element,FilterKey>,
    isIncluded: {title in title.hasSubstring(searchPhrase)}
) { smoothie in
        SmoothieRowView(smoothie: smoothie)
}
```
#### 4

```swift
FilteredList<Element, FilterKey, RowContent>(
    smoothies <as [Element],
    filterBy: \Element.title as KeyPath<Element,FilterKey>,
    isIncluded: { (title: FilterKey) -> Bool in title.hasSubstring(searchPhrase)}
) { smoothie in
        SmoothieRowView(smoothie: smoothie)
}
```

#### 5

```swift
FilteredList<Element, FilterKey, RowContent>(
    smoothies <as [Element],
    filterBy: \Element.title as KeyPath<Element,FilterKey>,
    isIncluded: {(title: FilterKey) -> Bool in title.hasSubstring(searchPhrase)}
) { (smoothie: Element) -> RowContent in
        SmoothieRowView(smoothie: smoothie)
}
```

上面这几步我们可以看到系统是如何将一个没有显示声明类型的方法在编译时一步步的添加对应类型的，下面我们在看下真正的类型推断步骤

#### 6

还是下面这段代码

```swift
struct SmoothieList: View {
    var smoothies: [Smoothie]
    @State var searchPhrase = ""
    
    var body: some View {
        FilteredList(
            smoothies,
            filterBy: \.title,
            isIncluded: {title in title.hasSubstring(searchPhrase)}
        ) { smoothie in
            SmoothieRowView(smoothie: smoothie)
        }
    }
}
```
首先我们第一个参数传入的是smoothies，我们已知这是一个Smoothie数组 所以可以推断出Element就是Smoothie

则上面的代码可以变成

```swift
FilteredList<Smoothie, FilterKey, RowContent>(
    smoothies <as [Smoothie],
    filterBy: \Smoothie.title as KeyPath<Smoothie,FilterKey>,
    isIncluded: {(title: FilterKey) -> Bool in title.hasSubstring(searchPhrase)}
) { (smoothie: Smoothie) -> RowContent in
        SmoothieRowView(smoothie: smoothie)
}
```
第二个参数传入的是 Smoothie.title 我们已知该字段是String类型，则上面的代码又可以变为

```swift
FilteredList<Smoothie, String, RowContent>(
    smoothies <as [Smoothie],
    filterBy: \Smoothie.title as KeyPath<Smoothie, String>,
    isIncluded: {(title: String) -> Bool in title.hasSubstring(searchPhrase)}
) { (smoothie: Smoothie) -> RowContent in
        SmoothieRowView(smoothie: smoothie)
}
```
第三个参数 是block中的返回值，很明显是一个SmoothieRowView，上面的代码又变为

```swift
FilteredList<Smoothie, String, SmoothieRowView>(
    smoothies <as [Smoothie],
    filterBy: \Smoothie.title as KeyPath<Smoothie, String>,
    isIncluded: {(title: String) -> Bool in title.hasSubstring(searchPhrase)}
) { (smoothie: Smoothie) -> SmoothieRowView in
        SmoothieRowView(smoothie: smoothie)
}
```

以上就是编译器进行类型推断的过程。

### 错误

假设我们在类型推断过程中出现错误怎么办？

假设我们的Smoothie结构体如下

```swift
struct Smoothie {
    var title: String
    var isPopular: Bool
}

```

比如 我们在上面的代码中推断搜索条件是根据title，那么如果我们把FilterKey类型写为isPopular呢？

```swift
FilteredList<Smoothie, Bool, SmoothieRowView>(
    smoothies <as [Smoothie],
    filterBy: \Smoothie.isPopular as KeyPath<Smoothie, Bool>,
    isIncluded: {(isPopular: Bool) -> Bool in isPopular.hasSubstring(searchPhrase)}
) { (smoothie: Smoothie) -> SmoothieRowView in
        SmoothieRowView(smoothie: smoothie)
}
```
这是后，编译器会发现 一个bool类型是无法调用`hasSubstring`方法的，因此这时候就会提示错误


### 错误收集

如果在类型推断过程中遇到了错误(比如上面描述的)，Swift会将这些错误收集，并尝试修复

- Records information about errors in source code
- Attempts to fix errors to continue the type inference algorithm
- Provides actionable error message based on collected information

如果你想看到更详细的信息可以通过设置 Xcode-Behaviors-EditBehaviors，这样你就可以看到，编译器提示的错误是在哪个位置进行类型推断时认为发生了错误

![设置](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/xocdenavigator.png)

例如

![错误](https://github.com/LeeWongSnail/AdvancedSwift/raw/main/res/errortracking.png)



### 参考文献

[Embrace Swift type inference](https://developer.apple.com/videos/play/wwdc2020/10165/)

