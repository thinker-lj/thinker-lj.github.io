---
title: 内存泄漏这件大事儿
date: 2016-02-26 22:20:19
tags: [ARC, Instruments, 内存优化, Leaks, 循环引用]
categories:  [技术实践]
---

> 你骄傲的和别人说，嘿，我写了个能让Windows崩溃的程序，他们会说“哥们儿，我装Windows系统的时候就免费带着了”
> <p align="right">—— Linus Benedict Torvalds</p>

-------
# & 引子
自打“小时候”接触C语言时，对内存处理机制的探究和兴趣，使我最终爱上了写代码这件事，最终走向了这条“不归路”……
本文当然不是内存基础知识科普，出于这个目的进来的朋友，我想Google更适合你！
今天主要聊聊iOS平台日常开发实践中，笔者经常遇到的内存泄露问题及处理策略。

-------

# & 平台相关
众所周知，早在5.0时iOS平台引入了自动引用计数(ARC)技术，这固然解放了不少在内存管理上面耗费的精力，可我始终认为，便捷和效率的提升带来的负面影响就是更多的人只为赶路，开始忽略背后的技术实现。这样的结局就是出了问题后的束手无策！
有了ARC，并不代表我们已经轻松到不需要了解内存管理的相关技术，恰恰相反，这是在MRC的基础上，又多了一种需要掌握的技术；同时也不能代表从此我们不会遇到内存相关的问题，实际上这要是本文探讨的核心问题。

-------

# & 值类型和引用类型
内存中需要管理的对象大致也就这两类：值类型和引用类型。值类型，例如int、枚举、结构体等基本数据类型(Swift 中集合类型也被实现为值语义，这带来诸多好处)都紧密排列位于栈中(这点并不绝对，例如较大结构体有置于堆中的可能)，这部分由系统统一管理，不在我们内存泄露的讨论范围。引用类型，也即是跟类扯上关系的一切实例，分配的内存在堆上面，需要我们手动管理，这部分体现在Foundation框架层就交由ARC来管理，当然通过桥接Core Foundation也可以借助ARC管理内存。我们所说的内存泄露，也主要是在引用类型的讨论上。<!-- more -->

-------

# & 内存泄露
下面从苹果官方文档定义的三种内存泄露情形，分别展开讨论：
## && Leaked memory
**应用中不再被引用，但是又不能被使用和释放的内存。**苹果提供的Leaks工具是可以检查出这部分内存泄露的，这部分内存泄露在MRC时代漏写release时很常见，C系函数中忘记free也同样会造成这种内存泄露。例如下面的场景：
```objc
CIContext *context = [CIContext contextWithOptions:nil];
CIImage *inputImage = [[CIImage alloc] initWithImage:image];
... ...
CGImageRef cgImage = [context createCGImage:result fromRect:inputImage.extent];
UIImage *resultImage = [UIImage imageWithCGImage:cgImage];
// CGImageRelease(cgImage); 
```
苹果始终在优化和扩展着Instruments系列的工具，Leaks工具在Xcode7之后也支持了真机调试。相信如果将Leaks工具检测列入日常开发的必走流程，这部分内存泄露便可以消灭在萌芽状态。
## && Abandoned memory
**应用中不再使用，但仍然被引用的内存。**很遗憾， Abandoned memory是无法通过Leaks工具发现的。这类内存泄露产生的原因，大多数都如下图所示的情形，也就是循环引用。
![](/images/14858506208778.jpg)
这只是造成循环引用的其中一种情况，block、delegate、NStimer等使用不当都会造成循环引用。解决方案就是从环中寻找恰当的点打破循环，针对图中的业务场景，在3处打破循环更为合理。下面对这类内存泄露的检测，提供些思路：
1. Allocations
这是Instruments工具集中的监测内存分配的工具，既然可以监测内存的使用，那通过针对特定界面的反复交互操作，观察Allocations的变化，是可以发现内存使用问题的。我们也可以通过代码模拟Allocations的类似功能，基本思路是通过hook对象生命周期的调用方法：alloc、dealloc、retain、release等，来监控对象的生命周期。但是，场景的重复操作实在过于繁琐，并没有完全解放双手。
2. 弱引用策略
试想，在对象创建时保存一个weak引用，并借助hook方法，进行全局监控，通过这种方式我们是可以监控整个应用的NSObject对象的，当然也可以缩小范围，只考虑Controller和View。当然，NSArray并不能保存weak引用，不过NSHashTable可以完美满足我们的需求！这种方式的确定时，无法在真正内存泄露时发出警告或者提醒，还是需要人为的预判和干预。
3. 开源库
针对内存泄露的监控，iOS平台已经有较为成熟的工具链，最为知名的当属Facebook开源的内存监控三件套：[FBRetainCycleDetector](https://github.com/facebook/FBRetainCycleDetector)、[FBAllocationTracker](https://github.com/facebook/FBAllocationTracker)、[FBMemoryProfiler](https://github.com/facebook/FBMemoryProfiler)，借助脚本和测试平台，可以实现一套完整的内存泄露检测工具链。

## && Cached memory
**应用中为了更好的性能，仍然被引用的内存。** 这类并不能算内存泄露，文档中已经讲得很清楚，其实就是内存缓存嘛！不过，在使用上面的内存泄露监测方法时，内存缓存带来的干扰和数据影响是不能被忽略的。在进行监测时记得对具体这类的所谓内存泄露添加例外，例如单例的使用等等。

-------
# & 最后
对内存泄露的重视程度，可以体现个人和团队的编程素养！更重要的，通过对内存泄露的把控和工具的不断打磨，可以积累更多实战中处理棘手问题的经验，工具习惯养成，专业态度的培养等等。





