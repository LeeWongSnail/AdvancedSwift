## Swift 内存管理

提到内存管理，我们就会想到OC中的ARC,与OC中相同，Swift中的内存管理也是ARC，依赖引用计数器来管理内存。那么Swift中的引用计数是如何存储的呢？我们都知道在OC中是会有一个全局的SideTables，内存存放当前生命周期中所有的弱引用对象以及他们的引用技术，同时包含引用技术为0时如何释放对象，弱引用表如何扩张和收缩。那我们就带着这些问题来了解下Swift中的内存管理吧。


### 什么情况下可能导致内存问题

在Swift中我们常遇到的可能导致内存问题的情况如下：

- 通知未移除(现在已经不会了)
- timer未移除
- block循环引用
- 代理
- KVO
- 对象间循环引用

这些情况在Swift中也是相同的，都可能会导致内存问题。

### 如何解决

方法也类似OC，使用weakSelf来解决 在Swift中写作`[weak self]`,即使用weak 来修饰self，





### 参考文献

[Swift进阶之内存管理 & Runtime!](https://zhuanlan.zhihu.com/p/376270278)
[Swift 内存管理](https://juejin.cn/post/7057832524663226405)
[Swift 内存管理](https://www.jianshu.com/p/07d49ddc98b1)
[详解Swift的内存管理](https://www.jb51.net/article/210839.htm)
[窥探Swift之数组安全索引与数组切片](https://cloud.tencent.com/developer/article/1018584)
[5 个让 Swift 更优雅的扩展](https://juejin.cn/post/7026271045652840461)
[]