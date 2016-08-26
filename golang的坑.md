使用golang开发新项目有一段时间了，现在逐渐对go有点失望了，但还好没有绝望。

最开始草草翻阅《go语音编程》（ 许式伟）时，最让我眼前一亮的其实并不是协程（这个已经听了太多太多...），而是其中对json的marshal和unmarshal，然后就了解到go可以对struct成员设置tag的特性。简单思考了一下，发现这个特性如果能够移植到我们现在的python代码中会让代码看起来更优雅一点。

随着项目的进行，对go的特性逐渐有了更深入的了解，然后越来越觉得之前有些看起来很棒的地方其实并没有想象中的那么好了。

# 现在开喷

* 由于golang是一门强类型语言，所以在json unmarshal时，如果struct的定义里不是指针，那么是无法区分到底是接收到了一个默认值还是json包中没有这个参数。这个其实问题也不算太严重，只是编程时需要小心的地方之一。

* 由于用惯了peewee，导致对golang的几个orm实现并不是很满意。在peewee中，model背后隐藏了一个dirty成员，这个成员可以记录你对那些列进行了修改，各种黑魔法（比如修改对象成员时可以通知对象）也使python实现的orm用起来十分自然、灵活。

* go虽然说为了保持语法的简单，使用了极少的关键字，然而实际上，为了实现某些功能，又隐藏了一些奇奇怪怪的东西在里边：比如iota，以`_test.go`结尾的测试代码，link的时候可以指定某些变量的值....不知道随着学习的深入，还会有哪些奇怪的东西在某一天忽然跳出来。不过想想应该会比c++少吧 :）

* 为了不让使用者随意使用go routine id做些事情，于是把go routine id给隐藏起来....官方的说法时，使用goroutine local storage时，万一忘了清理，就会很麻烦。这里有一个比较好用的gls库：[tylerb/gls](https://github.com/tylerb/gls)

* 最值得吐槽的就是官方库的实现了吧。log包几乎没有一点实用价值，相比于python logging包中的各种handler、formatter，你会深深觉得这个东西写的太随意。这里推荐一个觉得比较好用的log包：[inconshreveable/log15](https://github.com/inconshreveable/log15)。后续我大概有可能会根据这个实现一个log16吧 : )

* 对于go新颖的包导入方式....最终还是无法避免不同版本之间的兼容问题，于是搞了一个vendor。但这并不算完，曾经编译一个项目的时候提示找不到一个包，然后跑到对应的git仓库上一看，作者说"have moved to xxxx"... 

* 假设你的项目中用到了一个开源库A，并且用到了另外一个开源库B，其中B也用到了A，但是这个A是放置在B的vendor中的，那么在编译你的代码时，golang其实会把这两个当做不同的包编译进去的。好了，这其实不算是什么问题，**但是万一A用到了另外一个包C中的一个全局变量！！ GG**

# 说点好的

喷了这么多，golang里边的很多包有时候还是会给人带来很意外的惊喜的，从设计上或实现的功能上，让人眼前一亮。

* [golang.org/x/net/trace](https://godoc.org/golang.org/x/net/trace)：你可以通过创建一个trace对象，然后使用traceObj.LazyPrintf记录一些日志信息，同时启动默认的http server，这样你就可以在`http://default-server/debug/requests`这个页面查看所有正在活动中和已关闭的trace对象及其信息。

