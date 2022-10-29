## Swift KeyPath

在OC中如果我们想要使用KeyPath，我们必须硬编码，将对应keyPath以字符串的形式编写出来，这就存在可能出现的单词拼写问题。OC中给我们提供了KeyPath的方法，来避免这个问题。

下面我们通过这个结构体来讲解keyPath在Swift中的使用

```swift
struct Song {
    var name: String
    var genre: String
}
var favoriteSong = Song(name: "小幸運", genre: "抒情")
print(favoriteSong.name)
```

### 存取Property

```swift
let name = \Song.name
print(favoriteSong[keyPath: name])
```

`keyPath的写法: \ + 类型名 + property` 即 `\Song.name`

或者我们可以省略掉类型名

```swift
print(favoriteSong[keyPath: \.name])
```

### keyPath的类型

之前OC的时候我们实际上传入的是一个字符串，上面的例子我们看到在Swift中keyPath是这种形式`\Song.name`。那么这个写法下，keyPath是什么类型的呢？

![keypath](https://tva1.sinaimg.cn/large/008vxvgGgy1h7g80o3v82j30u00amaao.jpg)

WritableKeyPath 表示它是可以讀取和寫入的 KeyPath。

```swift
/// A key path that supports reading from and writing to the resulting value.
public class WritableKeyPath<Root, Value> : KeyPath<Root, Value> {
}
```

Song表示类型名，String表示要读取的property的类型

### 存取subscript

可以应用到数组或者字典中

数组

```swift
 var favoriteSongs = [
     Song(name: "小幸運", genre: "抒情"),
     Song(name: "漂向北方", genre: "饒舌")
 ]
 
 // 取数组中的第一个元素
 ///var song = favoriteSongs[keyPath: \[Song].[1]]
 var song = favoriteSongs[keyPath: \.[1]]
 print(song)
```

字典

```swift
 var favoriteSongDic = [
     "田馥甄": Song(name: "小幸運", genre: "抒情"),
     "黃明志王力宏": Song(name: "漂向北方", genre: "饒舌")
 ]
 ///var song1 = favoriteSongDic[keyPath: \[String: Song].["田馥甄"]]
 var song1 = favoriteSongDic[keyPath: \.["田馥甄"]]
 print(song1)
```

### KeyPath as Functions

对于下面这个结构体和数组

```swift
struct Book {
    var name: String
    var character: String
    var year: Int
}
var books = [
    Book(name: "愛麗絲夢遊仙境", character: "愛麗絲", year: 1865),
    Book(name: "湯姆歷險記", character: "湯姆", year: 1876),
    Book(name: "唐‧吉訶德", character: "唐‧吉訶德", year: 1605)
]
```

#### map

传统方法

```swift
let characters = books.map { $0.character }
```

利用keyPath

```swift
let characters = books.map(\.character)
```

#### filter

传统写法

```swift
let booksAfter1800 = books.filter { $0.year >= 1800 }
```

filter 的參數必須回傳 Bool，為了傳入 KeyPath，我們在 Book 裡定義 computed property after1800。
keyPath写法

```swift

extension Book {
  var after1800: Bool {
      year >= 1800
  }
}


let booksAfter1800 = books.filter(\.after1800)
```

#### compactMap

传统写法

```swift
struct Book {
    var name: String
    var character: String
    var year: Int
    var rabbit: String?
    
}
var books = [
    Book(name: "愛麗絲夢遊仙境", character: "愛麗絲", year: 1865, rabbit: "白兔"),
    Book(name: "湯姆歷險記", character: "湯姆", year: 1876),
    Book(name: "唐‧吉訶德", character: "唐‧吉訶德", year: 1605)
]
let rabbits = books.compactMap { $0.rabbit }
```

使用keyPath写法

```swift
let rabbits = books.compactMap(\.rabbit)
```

### 参考文献

[The power of key paths in Swift](https://www.swiftbysundell.com/articles/the-power-of-key-paths-in-swift/)

[How Swift keypaths let us write more natural code](https://www.hackingwithswift.com/articles/57/how-swift-keypaths-let-us-write-more-natural-code)

[KeyPath在Swift中的妙用](https://juejin.cn/post/6844903717511102472)
