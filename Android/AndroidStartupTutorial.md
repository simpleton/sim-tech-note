<!-- TOC depth:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [创业码农的一些建议](#)
	- [RxJava](#rxjava)
	- [MVVM & MVP](#mvvm-mvp)
	- [Image Loader](#image-loader)
	- [REST API](#rest-api)
	- [最佳实践](#)
	- [总结](#)
	- [参考文章](#)
<!-- /TOC -->

# 创业码农的一些建议
开篇先友情提示一下，此篇文章所谈论的技术点与微信无关，如有描述不准确的地方，也欢迎大家支出与讨论。笔者邮箱为 sunsj1231[at]gmail.com
作者去年离开微信，变成一个创业码农，期间也踩过一些坑，这里与大家分享一些我个人的经验。

微信整体的氛围还是很像创业公司的，快速、高效。但微信团队对技术的挖掘还是很深的。这一点是在创业公司很难做到的。创业公司更追求快速、稳定的做出功能，完成迭代。

下面给大家介绍一点我个人觉得很大的提高了我的开发效率的工具。

## RxJava
首先给大家安利[ReactiveX](http://reactivex.io/)，其中Android的核心实现为RxJava。

为了App不卡顿，我们会把所有耗时的操作（比如：网络访问、文件访问）放到Worker Thread中。但是Android本身的AsyncTask的设计个人觉得设计的十分糟糕，不但写出来的代码冗长，而且稍微复杂一些的多流操作就会写的完全无法维护（这里可以用Java本身的线程模式来实现）。而且肆意的开线程也会造成App的卡顿。这里本身最初的想法就是需要一个线程池，以Promise的方式对外提供接口。原先试用过facebook的开源方案[Bolts-Android](https://github.com/BoltsFramework/Bolts-Android)，这个库是parse的开源方案。后来有iOS的同事推荐Reactive的方案，于是就走上了Rx脑残粉的不归路。

首先Rx会大大减少你的代码量，这一点对“懒惰”的我们十分重要。
下面举2个平时开发都会遇到的问题来举例：

1. 搜索界面

	我们需要在用户输入完毕后第一时间显示搜索结果，由于这个需要请求后台，我们又不想用户每次输入的时候都去后台请求。并且总需要显示当前最新输入内容的结果，不能因为网络的原因产生乱序的结果。

	```java
	 RxTextView.textChanges(searchEditText)
        .compose(this.bindToLifecycle())
        .debounce(300, TimeUnit.MILLISECONDS)
        .switchMap(SearchService::searchFeed)
        .subscribe(
            feeds -> updateUI(),
            throwable -> RxUtil.handleError(throwable, activity)
        );
	```

	几句简单明了的代码，满足上述的需求，而且看起来十分明了简单。其中`.compose.(this.bindToLifecycle())`是为了防止内存泄露，`.debounce(300, TimeUnit.MILLISECONDS)`是表示间隔为300毫秒，使用switchMap是会停止之前发出的请求，防止脏数据重入。
	由于Android并不支持Java 8，所以我们需要Retrolambda，来支持lambda表达式。

2. 防止多次点击重入

	```java
	RxView.clickEvents(button)
        .throttleFirst(300, TimeUnit.MILLISECONDS)
        .subscribe(this::onButtonClick);
	```

## MVVM & MVP

![image](http://7tszlo.com2.z0.glb.qiniucdn.com/mvvm.pic.jpg)

关于MVVM我一直是拒绝的，因为一开始的几个Screen我是用硬套MVVM的模式来做的，虽然activity的代码十分简单，但是View和ViewModel都会写一些晦涩、重复的逻辑来保证数据绑定，这不符合D.R.Y.。后来发现google官方有一个[data-binding](http://developer.android.com/tools/data-binding/guide.html)的实现。感觉实现和prism十分类似。已经在最新的迭代中开始使用data-binding配合MVVM，具体可以参考一个例子:https://github.com/ivacf/archi

## Image Loader

整体APP的架构完成后，图片库也是对于APP十分重要的。笔者刚入职的时候，就是在照着google tutorial上的图片加载的例子写过一个ImageLoader，深深感到做一个高效的图片库还是有很大难度的。还好现在各路大神给了我们很多选择，下面3款笔者认为是可以选择的option:

	1. Picasso
	2. Glide
	3. Fresco

其中`Picasso`和`Glide`的接口十分接近，但是benckmark下来Glide的性能更好一些，并且支持更多格式的图片，我们现在使用的的是`Glide`，而`Fresco`的功能是这3个库中最强大的，且支持PJPG。但是他需要替换你的View，并且接口设计的不如上述2个库。笔者在3个多月以前用`Fresco`的时候，他在加载多张图片的时候偶尔会有显示不出的情况，不确定现在是否修复。

## REST API
关于REST API是一件几乎纯体力活，这里应当使用代码生成工具来帮助我们完成繁琐的工作。如果你的App追求极致的性能和流量，这里可以使用protocal buffer。这个有一个坑，就是PB原生的生成器生成的方法数非常多，会造成Android方法数64K的问题。可以使用Square开源的wire来降低方法数。

不过笔者的APP并没有使用二进制协议，使用了更容易调试的JSON。其中我们可以定义JSON Schema来描述协议，后台与客户端都可以拿这个schema来生成自己的Model和验证协议数据。Android中可以[jsonschema2pojo](https://github.com/joelittlejohn/jsonschema2pojo)来生成自己的Model代码，并且可以生成Parcelable代码(PS:这一部分可能还存在隐藏BUG，如果你在使用过程中有什么问题可以提issue或者直接联系笔者)。

关于REST API还有一个杀手级的库[Retrofit](https://github.com/square/retrofit)。Retrofit可以完美配合jackson+Rxjava来实现一个基于ReactiveX的REST Client。

```java
    @GET("/v2/feeds/search")
    Observable<List<FeedDetail>> searchFeeds(
            @Query("query") String query,
            @Query("tag") String tag,
            @Query("page") int page
    );
```

声明十分简单明了，具体可以去[retrofit](http://square.github.io/retrofit/)的官网了解更多。

## 最佳实践
关于最佳实践当然见仁见智，不过笔者还是推荐一些比较成熟的方案[android-best-practices](https://github.com/futurice/android-best-practices)，这个建议精读一下，里面的每一条都是别人踩过的坑总结来的，十分有价值。

虽然笔者一直是一个人默默开发，但是还是会遵守`git-flow`。每次多花一点点实践换来的分支的规整还是值得的。另外TJ开发的`git-extra`会有很多对github友好的命令，也值得推荐。

另外关于代码格式，也没有官方统一的方案，笔者这里推荐使用Square的[java-code-styles](https://github.com/square/java-code-styles),也可以自己fork做相应的修改。

另外强烈push设计的同学使用Sketch，这样不仅可以解放设计的同学在无尽的切图中，也可以让自己节约更多的时间。

## 总结


## 参考文章
[ReactiveX](http://reactivex.io/)

[RxJava](https://github.com/ReactiveX/RxJava)

[大神4篇](http://blog.danlew.net/2014/09/15/grokking-rxjava-part-1/)

[android-application-architecture](https://medium.com/ribot-labs/approaching-android-with-mvvm-8ceec02d5442#.suutwto9a)

[android-application-architecture](https://medium.com/ribot-labs/android-application-architecture-8b6e34acda65#.6qmzrqtdn)

[Improving UX with RxJava](https://medium.com/@diolor/improving-ux-with-rxjava-4440a13b157f#.21alo61m9)
