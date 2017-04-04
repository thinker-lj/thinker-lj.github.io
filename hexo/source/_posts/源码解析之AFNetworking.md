---
title: 源码解析之AFNetworking
date: 2016-04-15 23:47:51
tags: [HTTP, Runloop, NSOperation]
categories: 源码解析
---

> AFNetworking is a delightful networking library for iOS and Mac OS X. 
> It has a modular architecture with well-designed, feature-rich APIs that are a joy to use.

-------
# & 引子
毫无疑问，AFNetworking是个宝藏！用过它的人很多，但是如果你不满足于只是成为这个开源库的使用者，那就一起探索下它到底有多精彩！
开始之前有几点说明：
* AFNetworking在我写这篇文章时已经更新到3.x，最新版对URLConnection进行了阉割。个人认为这部分的细节依然干货满满，所以，下面的分析没有针对某个版本，着重在关键的几个重要实现。
* 本文假设阅读之前你已对 **HTTP(s)** 、**多线程编程**、**URL加载系统**有足够的了解。如果仍然云里雾里，停下来去了解下基础吧，文中也会在恰当处提供些较权威链接。
* 源码面前，了无秘密！没关注到的点，请阅读源码，同时也欢迎批评指正。

阅读源码，有个习惯，总会先梳理出框架的整体结构。这里从一个网络请求被AFNetworking接手到处理完成的角度，给出一个囊括主要功能类的结构图。当然，被3.x阉割的部分，也标注在它该有的位置。
![](/images/14744435828503.jpg)
整体来说，框架分为三层，请求被接手后首先是些编码序列化等处理。然后交给发送请求的管理类，这里也是最精彩核心的部分。最后，对返回结果再次进行格式化处理，至此AFNetworking任务完成。
下面，具体聊下每部分动辄上千行代码都做了什么…

-------

# & OperationManager
先从最核心的请求调度部分说起，无论是底层是采用 **NSURLConnection** 还是 **NSURLSession** ，每个请求都被封装成一个继承自 **NSOperation** 的对象，而OperationManager自然就是对多个请求的管理类。但具体到NSURLConnection和NSURLSession的实现上，又略有不同，下面具体说明：
## && URLConnection
再次强调，虽然 **NSURLConnection** 已经退出历史舞台并且AFNetworking的3.x版本已经剔除由NSURLConnection发的请求的封装类，但是，这里仍然会把它和NSURLSession同等看待。况且，二者的处理不仅存在差异，而且同样精巧！
### &&& AFURLConnectionOpration
先讨论一个关键问题：众所周知，NSURLConnection相关线程处理的异步网络请求，开始和回调需要在同一个线程，否则 **delegate** 无法被触发。这就涉及到如何做**线程保活**，等待回调触达。AFNetworking是这样做的：

```objc
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];

        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
```
这种方式首先出现在 **WWDC** 中，因此这里也只是借鉴，不是首创！
此 **Runloop** 与下面的全局网络请求线程做了绑定，所以NSURLConnection的网络请求，都是从这个线程发出并做回调处理的，最终做了默认回到主线程处理。

```objc
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });

    return _networkRequestThread;
}
```
下图描述了整个NSURLConnection请求的关键技术，AFURLConnectionOpration解决的主要难题。请求由不同的线程并发的发出，包装成operation，交付给上面提到的全局保活线程串行处理，然后NSURLConnection相关线程做真正的网络请求处理，将回调抛回全局线程，全局线程将处理的结果，继续抛回请求发出的线程。
![](/images/14744436276195.jpg)
关键问题说明白了，下面提几个关注到的技术点：
* Operation状态机
    为了保证NSOperation在加入到NSOperationQueue后的有效性，状态的控制通过 **KVO** 处理。继承自NSOperation的AFURLConnectionOpration针对操作的各个状态进行了统一管理，内部的状态统一由 **AFOperationState** 枚举类型属性state处理。处理的过程主要涉及到对状态转换的限制，枚举和NSOperation状态的一一对应等细节。
* 递归锁
    用锁的原因是为了保证能访问到成员变量的对外接口在多线程环境下的线程安全。锁的实现方式有很多种，这里之所以要采用NSRecursiveLock递归锁，是为了在内部无法确认请求的执行环境时，防止造成死锁的异常容错处理方式。
    <!-- more -->
* 多任务处理
    此类提供了同时处理多个operation的接口，关键技术是多个operation通过dispatch_group设置约束，等待全部回调处理完成统一抛给调用方。具体实现如下：
```objc
+ (NSArray *)batchOfRequestOperations:(NSArray *)operations
                        progressBlock:(void (^)(NSUInteger numberOfFinishedOperations, NSUInteger totalNumberOfOperations))progressBlock
                      completionBlock:(void (^)(NSArray *operations))completionBlock
{
   …
    __block dispatch_group_t group = dispatch_group_create();
    // group中操作处理完成后的后续处理
    NSBlockOperation *batchedOperation = [NSBlockOperation blockOperationWithBlock:^{
        dispatch_group_notify(group, dispatch_get_main_queue(), ^{
            if (completionBlock) {
                completionBlock(operations);
            }
        });
    }];
// 针对每个operation的处理
    for (AFURLConnectionOperation *operation in operations) {
        operation.completionGroup = group;
        void (^originalCompletionBlock)(void) = [operation.completionBlock copy];
        // 防止循环引用的处理
        __weak __typeof(operation)weakOperation = operation;
        operation.completionBlock = ^{
        // 每个operation完成后的处理
            __strong __typeof(weakOperation)strongOperation = weakOperation;
            dispatch_queue_t queue = strongOperation.completionQueue ?: dispatch_get_main_queue();
            dispatch_group_async(group, queue, ^{
                if (originalCompletionBlock) {
                    originalCompletionBlock();
                }

                …
                // 处理结束 离开group
                dispatch_group_leave(group);
            });
        };
        // 加入group
        dispatch_group_enter(group);
        // 设置batchedOperation的依赖，每个操作执行完成
        [batchedOperation addDependency:operation];
    }
    return [operations arrayByAddingObject:batchedOperation];·
}
```
    这里有个很重要的细节，group的 **dispatch_group_async** 处理被封装在 **batchedOperation** 这个operation中，然后设置了 **batchedOperation** 的依赖。其实不把group的最后处理封装进operation，也一样可以实现上述功能。但是作者采用了更优雅更安全的方式，这样的目的是为了把所有的操作都放在数组里传给上层，然后由同一队列进行处理。希望读这部分源码时，仔细品味下作者的巧妙严谨思路。
    
### &&& AFHTTPRequstOperation
这个继承自 **AFURLConnectionOpration** 的类主要完成的是断点续传的基础工作。断点续传需要注意HTTP请求头 **Range** 和 **If-Range** 。步骤是依托NSOutputStream获取到当前数据偏移量，然后设置请求的请求头Range，如果服务端存在 **ETag** 校验，将ETag设置到If-Range请求头中。这是断点续传的基础核心部分，其他如本地化等细节，为方便扩展，此类没有做处理。核心方法如下：

```objc
- (void)pause {
    [super pause];
    // 数据偏移量的获取
    u_int64_t offset = 0;
    if ([self.outputStream propertyForKey:NSStreamFileCurrentOffsetKey]) {
        offset = [(NSNumber *)[self.outputStream propertyForKey:NSStreamFileCurrentOffsetKey] unsignedLongLongValue];
    } else {
        offset = [(NSData *)[self.outputStream propertyForKey:NSStreamDataWrittenToMemoryStreamKey] length];
    }
    // 将偏移量设置到header中
    NSMutableURLRequest *mutableURLRequest = [self.request mutableCopy];
    if ([self.response respondsToSelector:@selector(allHeaderFields)] && [[self.response allHeaderFields] valueForKey:@"ETag"]) {
        [mutableURLRequest setValue:[[self.response allHeaderFields] valueForKey:@"ETag"] forHTTPHeaderField:@"If-Range"];
    }
    [mutableURLRequest setValue:[NSString stringWithFormat:@"bytes=%llu-", offset] forHTTPHeaderField:@"Range"];
    self.request = mutableURLRequest;
}
```
### &&& AFHTTPRequestOperationManager
这个只有二百来行的类是外部使用NSURLConnection进行网络请求，通常要用到的。它是对上面提到的Operation的友好封装。其实，早期版本请求并没有拆分的这样层次分明，现在这样的结构，这个封装管理类可以满足大部分的业务场景。碰到更多个性化的场景，还是需要直接调用上面提到的Operation来处理。
## && URLSession
 时间还要追溯到iOS 7 和 Mac OS X 10.9时代，Foundation URL 加载系统在那时被彻底重构了。上面提到了之前的做法：**NSURLRequest** 被传递给  **NSURLConnection** ，被委托对象（遵守以前的非正式协议 **<NSURLConnectionDelegate>** 和 **<NSURLConnectionDataDelegate>** ）异步地返回一个 **NSURLResponse** 以及包含服务器返回信息的 **NSData**。
 在那之后，**NSURLConnection** 替换成了 **NSURLSession**、**NSURLSessionConfiguration** 以及 **NSURLSessionTask** 的 3 个子类：NSURLSessionDataTask，NSURLSessionUploadTask，NSURLSessionDownloadTask。这里更多的细节可以通过官方文档和文末的扩展阅读深入了解。下面我们要讨论的是AFN如何借助 **NSURLSession** 完成完整的网络请求。
### &&& AFURLSessionManager
上面聊NSURLConnection相关问题时，很大篇幅都在讨论线程保活策略。甚至更进一步讲，它成了核心，也正是因为线程保活，才使得 **AFURLConnectionOpration** 显得有些繁杂。但这一切，在由NSURLSession发起的请求中，得到了充分的简化。因为在session的初始化时，允许设置回调队列和回调target。下面，我们慢慢来体会：
#### &&&& 初始化
初始化时主要做了下面几件事儿：
1. 设置请求的 **AFURLConnectionOpration** ，支持接口外传；
2. 设置请求回调处理队列，注意此处使用的**串行**队列；
3. 初始化NSURLSession对象，传入 **delegate 宿主**和2中提到的执行delegate的队列；
4. 数据解析和安全策略的相关初始化设置；
5. 锁的初始化，后面会在竞争资源中使用；
6. 针对不同的 **NSURLSessionDataTask** 设置不同的delegate处理方式；
7. 监听任务开始和暂停通知，这里有个细节：通过hook任务开始和暂停，加入对应通知；

```objc
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
   … …
   // 1
    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }
    self.sessionConfiguration = configuration;
    // 2
    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;
    // 3
    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];
    // 4
    self.responseSerializer = [AFJSONResponseSerializer serializer];
    self.securityPolicy = [AFSecurityPolicy defaultPolicy];
    … …
    // 5
    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;
    // 6
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task completionHandler:nil];
        }
    … …
    }];
    // 7
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidResume:) name:AFNSURLSessionTaskDidResumeNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidSuspend:) name:AFNSURLSessionTaskDidSuspendNotification object:nil];
    return self;
}
```

#### &&&& AFURLSessionManagerTaskDelegate
这是个值得关注和效仿的类，主要体现在对代理的处理策略。它是NSURLSession的代理封装类，如果只考虑可用性，其实完全可以在AFHTTPRequestOperationManager直接实现相关代理。这里将代理的实现从主类中抽离，并且通过NSProgress提供任务进度功能，很有效的减少了主类的复杂度，更进一步使功能更加模块化。
主类和代理类的衔接，依靠下面简短的构造和设置：
```objc
    // 构造AFURLSessionManagerTaskDelegate
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    // 主要用于在代理类中获取主类属性
    delegate.manager = self;
    // 请求完成的回调处理
    delegate.completionHandler = completionHandler;
    // 标识task
    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    // 下面三行是 [self setDelegate:delegate forTask:dataTask]的展开
    [self.lock lock];
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [self.lock unlock];
```
在代理类中值得一提的，还有对NSProgress的应用。它是和NSURLSession同时在iOS7中被引入的，目的是建立一个标准机制来报告长时间运行的任务进度。[这里](https://developer.apple.com/reference/foundation/progress)是苹果对它的详细介绍和使用范例。
在此处作者主要是将代理类的progress与dataTask相互关联，然后赋值给外部传进来的NSProgress对象，方便业务方对进度的把控和任务的处理；部分细节如下：
```objc
- (void)addDelegateForUploadTask:(NSURLSessionUploadTask *)uploadTask
                        progress:(NSProgress * __autoreleasing *)progress
               completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    ……
    // 计算获取的目标数据量
    int64_t totalUnitCount = uploadTask.countOfBytesExpectedToSend;
    if(totalUnitCount == NSURLSessionTransferSizeUnknown) {
        NSString *contentLength = [uploadTask.originalRequest valueForHTTPHeaderField:@"Content-Length"];
        if(contentLength) {
            totalUnitCount = (int64_t)[contentLength longLongValue];
        }
    }
    // 设置progress的totalUnitCount
    if (delegate.progress) {
        delegate.progress.totalUnitCount = totalUnitCount;
    } else {
        delegate.progress = [NSProgress progressWithTotalUnitCount:totalUnitCount];
    }
    // 关联task的暂停
    delegate.progress.pausingHandler = ^{
        [uploadTask suspend];
    };
    // 关联task的取消
    delegate.progress.cancellationHandler = ^{
        [uploadTask cancel];
    };
    // 设置业务方传入对象
    if (progress) {
        *progress = delegate.progress;
    }
    ……
}
```
有真实场景应用进度反馈经验的同学，可能注意到这个进度反馈并不是很准确。这并不是AFN的疏漏，因为真实的进度是很难获取的，上层的HTTP请求数据会在TCP层的 **Send buffer** 中缓存，然后才会真正上传给服务器。因此，在数据量较小的进度反馈中，你会看到进度条一闪而过。这就可以理解，类似微信的假进度条，真是不得已为之。
#### &&&& 几个技术细节
在NSURLSession的引导下，这个类的实现其实一切都显得很自然，理顺主流程的前提下，下面聊几个注意到的技术细节：
* 任务获取中利用信号量巧妙将异步转换成同步：
```objc
// 创建信号量
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        … …
        // 执行完毕信号
        dispatch_semaphore_signal(semaphore);
    }];
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
```
* 通过swizzle技术为NSURLSessionTask的resume和suspend添加通知：
```objc
        … …
        IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
        Class currentClass = [localDataTask class];
        // 覆盖不同系统版本的NSURLsessionTask实现差异
        while (class_getInstanceMethod(currentClass, @selector(resume))) {
            Class superClass = [currentClass superclass];
            IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
            IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
            if (classResumeIMP != superclassResumeIMP &&
                originalAFResumeIMP != classResumeIMP) {
                [self swizzleResumeAndSuspendMethodForClass:currentClass];
            }
            currentClass = [currentClass superclass];
        }
        … …
```
    要理解上面这段代码，首先要明确NSURLSession是个类簇，也就是设计模式经常提到的抽象工厂模式产物。而swizzle是需要明确方法真正的所属类，因此便有了上面的循环处理，去hook所有的resume和suspend实现。

### &&& AFHTTPSessionManager
这个类就没太多需要细聊得内容了，通过继承自AFURLSessionManager，提供更加友好易用的接口；

-------

# & Serialization
下面聊下框架图中的第一部分和第三部分，请求在发出之前的配置工作由**AFURLRequestSerialization**负责，回调后的数据处理工作由**AFURLResponseSerialization**负责。这两部分难度不大，主要是细节更多一些。
## && AFURLRequestSerialization
首先，这是个协议，一个针对HTTP请求进行处理的协议。
提到这个协议，有个细节，这个框架中需要序列化的内容，都遵循的是**NSSecureCoding**协议，相比于**NSCoding**，它提供安全序列化，保证结果的类型可信赖性，可以防止数据被篡改损坏造成的decode时产生异常情况。
它要求代理类实现下面的方法：
```objc
- (nullable NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(nullable id)parameters
                                        error:(NSError * __nullable __autoreleasing *)error
```
这个方法主要是通过配置参数，构造**NSURLRequest**对象，用于后续的请求。
### &&& 基类：AFHTTPRequestSerializer
**AFHTTPRequestSerializer**是请求处理的核心基类，实现了上面提到的**AFURLRequestSerialization**协议，它主要做了下面几件事：
* HTTP请求头设置
    此类对外提供了设置HTTP header的方法：
    **- (void)setValue:(nullable NSString *)value forHTTPHeaderField:(NSString *)field**
    在初始化方法中也对HTTP 请求头进行了部分设置，如**User-Agent**，这里值得一提，UA(User-Agent)具体规则参考[rfc](https://tools.ietf.org/html/rfc7231#section-5.5.3)这个请求头字段作用非常大，它涉及界面展现和数据分析等，这里做了处理，外部不需要再重复设置如下内容：
```objc
userAgent = [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?:
 [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: 
 [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[UIDevice currentDevice] model], [[UIDevice currentDevice] systemVersion], [[UIScreen mainScreen] scale]];
```
    另外还有些请求认证信息**Authorization**等通过外部接口来进行的相关设置。
* 请求的**query** 处理：
    查询参数相关处理主要是通过**AFQueryStringPair**这个类，对应着**field**和**value**两个属性的操作，主要方法如下：
```objc
- (NSString *)URLEncodedStringValue {
    if (!self.value || [self.value isEqual:[NSNull null]]) {
        return AFPercentEscapedStringFromString([self.field description]);
    } else {
        return [NSString stringWithFormat:@"%@=%@", AFPercentEscapedStringFromString([self.field description]), AFPercentEscapedStringFromString([self.value description])];
    }
}
```
    目的是将field和value格式化成请求问号后面的等号连接形式。外层是一个递归函数：**AFQueryStringPairsFromKeyAndValue**，它将传入的集合递归处理，调用上面的格式化方法处理成标准的**query string**。    
* 其他request相关属性设置
    这部分对外暴露了**超时时间**、**HTTP管线化**、**缓存策略**等设置属性，然后通过KVO映射到**mutableObservedChangedKeyPaths**属性，从而影响request的构造。核心代码如下：
```objc
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(__unused id)object
                        change:(NSDictionary *)change
                       context:(void *)context
{
    if (context == AFHTTPRequestSerializerObserverContext) {
        if ([change[NSKeyValueChangeNewKey] isEqual:[NSNull null]]) {
            [self.mutableObservedChangedKeyPaths removeObject:keyPath];
        } else {
            [self.mutableObservedChangedKeyPaths addObject:keyPath];
        }
    }
}
```
### &&& 子类任务及其他相关功能
AFHTTPRequestSerializer已经做了request相关的所有工作，下面是更加丰富的功能。主要有以下几方面：
* 子类数据格式扩展
**AFJSONRequestSerializer**和**AFPropertyListRequestSerializer**继承自上面提到的**AFHTTPRequestSerializer**，它们在基类基础上，负责实现在**AFURLRequestSerialization**协议中的唯一方法，来提供JSON数据格式和plist数据的格式化，并设置到HTTPBody中。
* 构造Multipart协议类型请求
    这部分篇幅很长，但目标单一。主要通过类**AFMultipartBodyStream**依据Multipart协议([rfc1867](https://www.ietf.org/rfc/rfc1867.txt))，做表单数据拼接。主要满足非key=value形式的文件实体的传输。
    这里有个技术点，针对较大文件的处理方式。为了避免大文件一次性读取和**AFStreamingMultipartFormData**构造造成的内存暴涨，采用了文件读取分片和数据上传分片的策略。支撑这一切的是**AFStreamingMultipartFormData**和**AFMultipartBodyStream**自定义数据格式。
    
## && AFURLResponseSerialization
这部分内容结构类似前面讨论的请求处理，同样**AFURLResponseSerialization**是个协议，主要目的是对请求返回数据的序列化规范。下面是它的唯一需要代理类实现的方法：
```objc
- (nullable id)responseObjectForResponse:(nullable NSURLResponse *)response
                           data:(nullable NSData *)data
                          error:(NSError * __nullable __autoreleasing *)error
```
另外结构类似的一点是返回数据处理这部分也是由核心基类和个性化数据需求子类组成。具体内容如下：
### &&& 基类：AFHTTPResponseSerializer
内容不多，我们按照源码顺序，分三部分说明一下：
1. 初始化方法
    初始化方法主要对对外暴露的属性赋值默认值，另外值得注意的是**acceptableContentTypes**交给了子类处理。
```objc
- (instancetype)init {
    self = [super init];
   …

    self.stringEncoding = NSUTF8StringEncoding;
    //acceptableStatusCodes 设置 200 ~ 299 HTTP有效响应的范围
    self.acceptableStatusCodes = [NSIndexSet indexSetWithIndexesInRange:NSMakeRange(200, 100)];
    // 子类根据不同数据类型自行处理
    self.acceptableContentTypes = nil;
    
    …
}
```
2. 响应有效性判断
    主要判断流程分为以下三步：
    * 设置默认有效，后面只处理无效情况；
    * 根据初始化的acceptableContentTypes 判断MIME 媒体类型是否合法；
    * 根据初始化的acceptableStatusCodes 判断状态码是否有效；
3. **AFHTTPResponseSerializer**代理
因为基类不涉及数据类型处理，这里直接调用响应有效性判断对数据的处理结果。

### &&& 子类：数据类型序列化扩展
继承自**AFHTTPResponseSerializer**数据处理子类主要针对以下几种格式数据：**JSON**、**XML**、**Image**、**Plist**。
这部分较为简单，主要就是借助系统API实现数据的指定格式化行为，下面说几个细节：
* AFNetworking默认设置的回调数据解析格式是最常用的JSON；
* XML格式有两个类，分别对应OS X和iOS不同系统解析器；
* 图片操作最系统的方法进行了加锁改造，同时支持<1M的图片数据解码；
* 支持数据类型不确定时传入多个数据类型进行序列化操作，对应类：**AFCompoundResponseSerializer**，就是简单的遍历，直到序列化成功返回。

-------

# & Security
安全无小事！但是，当今国内互联网环境，大多数为安全所做的推动，都只是块**遮羞布**，根本没有达到**保护膜**的程度。
Apple 凭借得天独厚的生态优势，在技术推动上所做的贡献，是值得称赞的。相信很快App Store会对HTTPS有新的推进举措。也是出于这个原因，AFNetworking在2.x版本，通过重构，全面支持了HTTPS，证书验证部分主要就体现在下面要聊的**AFSecurityPolicy**。
这部分主要是对系统的**Security**的友好封装，没有复杂的逻辑，先看下几个细节：
* 实现了自动读取bundle中得证书文件，所以如果有本地内置证书需要的话，导入证书即可；
* 注意上面提到的证书验证模式的选择；
* 对于自建证书，通过**allowInvalidCertificates**属性来设置；
* 如果需要做域名限制，通过**validatesDomainName**属性设置；

## && 证书绑定选项
先看下这个枚举：
```objc
typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    AFSSLPinningModeNone,    // 无证书绑定 验证系统信任证书
    AFSSLPinningModePublicKey,    // 内置证书 只验证公钥
    AFSSLPinningModeCertificate,    // 内置证书 验证证书是否有效且是否一致
};
```
通过方法**+ (instancetype)defaultPolicy**初始化时默认使用的是**AFSSLPinningModeNone**，也就是客户端不在本地内置server证书，只会去验证系统信任的证书。考虑到与同一客户端打交道的server应该并不多，最好还是将证书内置，做server证书的验证，甚至应该实现HTTPS**双向认证**。

## && 初始化
提供了两个初始化方法，最终都会触发的**+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode**方法。在初始化中，主要完成下面两件事：
* 提供证书绑定的模式设置入口，并从Bundel中自动导入证书：
```objc
+ (NSArray *)defaultPinnedCertificates {
    // 静态集合
    static NSArray *_defaultPinnedCertificates = nil;
    static dispatch_once_t onceToken;
    // 处理一次
    dispatch_once(&onceToken, ^{
        NSBundle *bundle = [NSBundle bundleForClass:[self class]];
        NSArray *paths = [bundle pathsForResourcesOfType:@"cer" inDirectory:@"."];

        NSMutableArray *certificates = [NSMutableArray arrayWithCapacity:[paths count]];
        for (NSString *path in paths) {
            NSData *certificateData = [NSData dataWithContentsOfFile:path];
            [certificates addObject:certificateData];
        }

        _defaultPinnedCertificates = [[NSArray alloc] initWithArray:certificates];
    });

    return _defaultPinnedCertificates;
}
```
    这里涉及一些细节及流程上的优化，**_defaultPinnedCertificates**是存放证书的集合，但只做一次初始化处理。因为证书是在每次请求都是要做验证的，但是没有必要每次重复导入。另外一点，要理解作者的这个思路，**自动**做起来也许很简单，但是首先要具备的是这个意识。
* 获取证书的公钥：
```objc
- (void)setPinnedCertificates:(NSArray *)pinnedCertificates {
    //借助NSOrderedSet 数组去重并导出
    _pinnedCertificates = [[NSOrderedSet orderedSetWithArray:pinnedCertificates] array];
    …
   // 读取所有证书的公钥信息并保存
   for (NSData *certificate in self.pinnedCertificates) {
       id publicKey = AFPublicKeyForCertificate(certificate);
      …
      }
}
```
    注意一个细节，在早期版本中是没有借助**NSOrderedSet**去重操作的，算是对证书遍历的一个小优化吧。数组去重有很多种方式，像作者这样利用NSSet的特性来处理，最简单优雅。
    
## && 证书验证
终于到了最重要的环节，证书的验证主要分为以下几步：
1. 判断是否传入域名，并处理特殊情况：如果传入域名，不能选择**SSLPinningMode**模式同时证书必须要存在。
```objc
if (domain && self.allowInvalidCertificates 
&& self.validatesDomainName 
&& (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)) {
return NO；
}
```
2. 创建安全策略**SecPolicyRef**，设置policy：
```objc
NSMutableArray *policies = [NSMutableArray array];
    // 如果验证域名
    if (self.validatesDomainName) {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {// 否则创建X509标准SecPolicyRef
        [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
    }
    // 设置基于的安全策略
    SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);
```
3. 处理系统信任证书的验证结果：
```objc
// AFSSLPinningModeNone模式
if (self.SSLPinningMode == AFSSLPinningModeNone) {
        // 允许不合法证书或者验证通过
        return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
    } else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
        return NO;
    }
```
4. 处理公钥验证和证书验证模式，这部分逻辑类似，递归对比证书或公钥。

-------
 
# & Reachablity
到这里我们几乎以及啃完了AFN所有的硬骨头！下面的会相对轻松些了。
对**SystemConfiguration**模块的封装，有很多个现成库，实现也大都类似。下面我们从网络监控的流程角度，一起看下iOS平台如何完成的网络监控。
## && 初始化
这是个单例，在我的[另外一篇文章](http://thinker-lj.com/我为什么要慎重使用单例/)中，详细讨论了单例的适用场景，其中也提到了网络监控，这个功能用单例再合适不过了。在初始化方法中，主要完成下面几件事：
* 通过**domain**或者**address**生成**SCNetworkReachabilityRef**句柄；
* 通过SCNetworkReachabilityRef初始化**AFNetworkReachabilityManager**；
* 设置**networkReachabilityStatus**网络状态为未知类型；

## && 开启监控
这是整个网络监控的核心部分，主要按顺序做了下面几件事：
1. 先关闭stopMonitoring（从main RunLoop 中移除） ，目的是防止被多次开启。
```objc
- (void)stopMonitoring {
   …
    // 从主线程的RunLoop中移除网络监控
    SCNetworkReachabilityUnscheduleFromRunLoop(
    (__bridge SCNetworkReachabilityRef)self.networkReachability, 
    CFRunLoopGetMain(), 
    kCFRunLoopCommonModes);
}
```
2. 创建并设置网络状态改变的回调：
```objc
// 留意为避免循环引用，这里的weakSelf、strongSelf 最佳实践
__weak __typeof(self)weakSelf = self;
    AFNetworkReachabilityStatusBlock callback = ^(AFNetworkReachabilityStatus status) {
        __strong __typeof(weakSelf)strongSelf = weakSelf;
        strongSelf.networkReachabilityStatus = status;
        if (strongSelf.networkReachabilityStatusBlock) {
            strongSelf.networkReachabilityStatusBlock(status);
        }

    };
```
3. 在主线程的Runloop 开启网络监控；
4. 在**dispatch_get_global_queue**队列中，获取当前网络状态并做网络状态改变的后续处理；
```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),^{
        SCNetworkReachabilityFlags flags;
        if (SCNetworkReachabilityGetFlags((__bridge SCNetworkReachabilityRef)networkReachability, &flags)) {
            // 网络状态改变的后续处理
            AFPostReachabilityStatusChange(flags, callback);
        }
    });
```

## && 网络状态改变的处理
上面提到了网络状态改变的后续处理，其实主要就是发送通知，切记通知一定要在主线程发送及监听，这是为了避免通知发送方和监听方不在同一线程导致无法触发的问题。
```objc
static void AFPostReachabilityStatusChange(SCNetworkReachabilityFlags flags, AFNetworkReachabilityStatusBlock block) {
    // 这里通过KVO处理网络状态的改变
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusForFlags(flags);
    // 回到主线程处理通知
    dispatch_async(dispatch_get_main_queue(), ^{
        …
        NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
        NSDictionary *userInfo = @{ AFNetworkingReachabilityNotificationStatusItem: @(status) };
        [notificationCenter postNotificationName:AFNetworkingReachabilityDidChangeNotification object:nil userInfo:userInfo];
    });
}
```

网络监控这部分虽然不是很复杂，但是深挖依然可以找到作者很多最佳实践，比如**static**函数、**__clang_analyzer__**、**FOUNDATION_EXTERN**还有KVO的键依赖等等。
唯一感到遗憾的是，这里没有提供2G/3G/4G等网络类型的细化，不过依然可以通过其他方式拿到更具体的网络类型，我比较喜欢的是直接通过KVC读取状态栏信息，简单粗暴！

-------

# & 最后

-------
扩展阅读：
[From NSURLConnection to NSURLSession](https://www.objc.io/issues/5-ios7/from-nsurlconnection-to-nsurlsession/)


 


