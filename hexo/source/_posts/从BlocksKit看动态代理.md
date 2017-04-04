---
title: 从BlocksKit看动态代理
date: 2017-01-22 14:24:52
tags: [BlocksKit, Delegate, Protocol, Runtime]
categories: 源码解析
---
> 你试过倒立或者跑起来看周围的世界吗？

-------

# & 引子
之前写过一些源码分析的文章，例如[AFNetworking](http://thinker-lj.com/%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%B9%8BAFNetworking/)这篇大概敲了两万多字，花了差不多工作之余的将近一周时间。外加在团队内做了分享，算下来的时间投入是个不小的数字。当然，这个时间值得投入，可在我看来，并没有利用到最大化。[帕雷托法则（二八定律）](https://zh.wikipedia.org/wiki/%E5%B8%95%E9%9B%B7%E6%89%98%E6%B3%95%E5%88%99)告诉我们80%的结果取决于20%的原因，我相信很多经典的开源库可以做到字字珠玑，但是真正核心的内容和思想精髓还是那需要花费我们80%的时间去体会的那20%！因此，我做了一个重要的决定：再写源码分析的文章，不再面面俱到，具体细节会在开源的源码注释中体现，这里我只需要表达出来通过阅读源码后的真正印象深刻的，残留在脑海中的，内化后的感受就可以了。下面，就以此篇做个尝试……

-------

# & 走马观花
在iOS的开源社区五千多的 star ，[BlocksKit](https://github.com/zwaldowski/BlocksKit)其实算不上特别著名。之所以注意到它，是在前段时间研究 **Block** 的内部实现时，成了延伸阅读，竟然发现还很有趣。它为很多常用类提供了更加便捷和友好的附带block参数的接口，当然这部分实现很简单，简单举个例子：

```objc
- (NSArray *)bk_map:(id (^)(id obj))block {
	NSParameterAssert(block != nil);
	
	NSMutableArray *result = [NSMutableArray arrayWithCapacity:self.count];
	[self enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
		id value = block(obj) ?: [NSNull null];
		[result addObject:value];
	}];

	return result;
}
```
这是个为集合类型提供类似 map 高阶函数的接口，内部直接调用了系统 API ，在合适的地方执行block即可；源码中大部分的实现都类似这种处理策略，唯有针对代理的处理方式让人眼前一亮，这便是这个库的核心了！<!-- more -->

-------

# & 大胆假设
先抛开 BlocksKit 关于动态代理的实现，我们先简单聊聊代理。**代理**算是对象结构型模式中比较重要的一种设计模式了，具体到 Objective-C 中，我们通常会通过 **Protocol** 来定义一组描述某种功能的接口协议，由委托类处理协议流程，然后由被委托类对协议做具体个性化实现。这在我的另外一篇文章 [Swift之面向协议编程](http://thinker-lj.com/Swift%E4%B9%8B%E9%9D%A2%E5%90%91%E5%8D%8F%E8%AE%AE%E7%BC%96%E7%A8%8B/)对 Swift 环境下的相关新的特性做了详细阐述。同时，业界也有很多关于代理类如何造成循环引用的讨论，以及讨论代理、协议、block区别的文章。可见，代理这个话题并不新鲜，但是这些讨论都停留在代理固有特性的细节层面，我们今天做个大胆的假设：我们可否充当动态的代理类，在 block 中去做协议接口的动态实现……

-------

# & 渐入佳境
试想，在我们自己定义的类中，该如何实现这种动态代理效果？其实很简单，下面就看个例子：我们先简单定义一个协议：
```objc
@Protocol XXClassDelegate<NSObject>
- (void)fetchContent:(NSString)content target:(id)target;
@end
```
然后在委托类的合适地方，调用代理方法，其中这里包括动态调用的版本：
```objc
@interface XXClass: NSObject <XXClassDelegate>
@property (nonatomic, weak) id <XXClassDelegate> delegate;
@property (nonatomic, copy) void(^bk_fetchContentWithTargetBlock)(NSString *, id *);
@end

@implementation XXClass
- (void)fetchHandle {
	if([_delegate respondsToSelector:@selector(fetchContent:target:)]) {
		// 原有代理模式
		[_delegate fetchContent:nil target:self]];
	} else if (bk_fetchContentWithTargetBlock) {
		// block 分支：需要代理类动态实现
		[self _bk_fetchContentWithTargetBlock(nil, self)];
	}
}
@end
```
这样，我们只需要在代理类中需要的地方实现bk_fetchContentWithTargetBlock就可以达到之前协议的效果了。其实，这只不过是将原本需要实现协议做的事情，替换成了 block 实现罢了。我们之所以能这样做，是因为操作协议的类是我们自己定义的，如果是系统 API，比如 UITableView，就无法这样做了，因为我们修改不了UITableView的具体实现。

-------

# & 拨云见日
针对我们无法修改的类，我们该如何去动态代理呢？有这么一个玩笑，在软件系统中，没有什么是增加一个中间件解决不了的，如果有，那就增加两个！虽然是玩笑，却给我们提供了思路！
我们知道，协议是委托类和被委托类之间的桥梁，既然现在要动态代理，那这个桥梁就无法完整搭建，这时我们可以在二者之间修条辅路，这个辅路的角色，动态的充当被委托类去实现协议，而在其内部去执行真正被委托类定义的 block 。**BlocksKit** 的精髓就在这里，下面我们以 **UIWebView** 为例来简单以图示意：
![](/images/14863431193011.jpg)

原本两层的委托与被被委托关系，通过增加针对性的中间层，利用 runtime 替换原有被委托类，在新委托类中实现协议。这就是整体的动态代理实现思路，当然，中间存在很多细节，针对多个系统类需要设计集散分发中心，具体协议的查找、delegate 动态替换、block 与原方法的映射缓存等等都要统一在这里完成。但是大部分的细节都是在整体思路的规划上，一步步细化并可以迎刃而解的问题，这里不再做具体阐述。

-------

# & 最后
严格意义上讲，这并不算是真正的源码分析，更详细的代码行级注释会在后续整理完成后的开源项目中体现。回到动态代理的讨论上，在实际应用场景中，我并不赞同这种方式的大范围使用，毕竟系统框架设计者完全有能力将 delegate 用 block 的形式定义。之所以为之，其实 delegate 相对来说，虽然更重一些，需要接口实现，方法分离也会带来临时数据的处理，代码也很不清晰，但是当需要定义接口不止一个，同时可以被归为一组，又会被多次调用的场景，delegate 就更有优势了！总之，任何技术都有它适用的场景，千万不要犯一味炫技，拿着锤子找钉子的偏激错误。


