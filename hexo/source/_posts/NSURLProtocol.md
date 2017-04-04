---
title: 你真的会用NSURLProtocol？
date: 2016-05-01 15:22:55
tags: [NSURLProtocol, hook]
categories: 奇技淫巧
---

![题图来自最近玩的一个小游戏](/images/NSURLProtocol_p0.jpeg)

-------

> NSURLProtocol 或许是URL加载系统中功能最强大但同时也是最晦涩的部分了…
>
> <p align="right">—— [Mattt Thompson](http://nshipster.cn/authors/mattt-thompson/)</p>												

-------

# & 引子
网上针对NSURLProtocol的讨论不仅少之又少，而且在仅有的资料中还有不少纰漏。显然大多停留在文档阅读或者demo阶段，并没有在工业级项目中应用。这也是我写这篇文章的原因，还是一贯原则：实践出真知！所以本文主要探讨笔者在项目中对NSURLProtocol的应用，旨在填坑和思维发散…

-------

## && 谈几点经验：

- NSURLProtocol处在iOS开发中的核武器级别，一旦**registerClass**，影响的是整个项目的URL加载系统；
- 鉴于上条原因谈到的影响范围，在工业级项目中使用时记得周知QA，做好全面回归覆盖case；
- 处理好**request**标记，防止死循环；
- 尽可能全部处理**NSURLProtocolClient**，尤其是**重定向**的方法；
- 复杂业务场景下，谨慎处理多个NSURLProtocol子类的注册和继承嵌套；
- 可处理**NSURLConnection**、**NSURLSession**、**UIWebView** 对**WKWebView**无法截获；

> 上面谈到NSURLProtocol功能强大，接下来只聊已经实践过的几个case，希望可以抛砖引玉…

<!-- more -->

-------

# & 一、监控所有接口的响应时间
步骤如下：
1. 在**startLoading**方法中修改**request**，在**HTTPHeader**中增加请求发起的时间；

   ```objc
   - (void)startLoading {
       NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
       [NSURLProtocol setProperty:@YES forKey:kURLProtocolHandledKey inRequest:mutableReqeust];
       [mutableReqeust setValue:[NSString stringWithFormat:@"%@", @([NSDate date].timeIntervalSince1970)] forHTTPHeaderField:@"requestTime"];
       self.connection = [[NSURLConnection alloc] initWithRequest:mutableReqeust delegate:self startImmediately:YES];
   }
   ```

2. 在**connectionDidFinishLoading**中进行时间的计算；
   ```objc
   - (void) connectionDidFinishLoading:(NSURLConnection *)connection {
       [self.client URLProtocolDidFinishLoading:self];
     // 取出header中的请求发起时间，计算出响应时间并记录或上报；
       [self p_reportLogWithConnection:connection error:nil];
   }
   ```

1. 考虑到hook的是所有的网络请求，加了接口过滤功能：
   ```objc
   // 过滤无效请求
   + (BOOL)filterInvalidHost:(NSString *)host {
       if (!host) {   
           return YES;
       }
       __block BOOL invalid = YES;
       static NSArray *filterHostArr;
     // 合法的接口Host
       filterHostArr = @[@"XXX.com"……];
       [filterHostArr enumerateObjectsUsingBlock:^(NSString * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
           if ([host containsString:obj]) {
               invalid = NO;
               *stop = YES;
           } 
       }];
       return invalid;
   }
   ```
2. 因为整个过程不涉及UI刷新操作，最好另起线程处理；
3. 真正项目中实现的功能，远远不止一个响应时间，通过这个思路，可以实现很多类似功能，我们已经通过此途径搭建了一个接口监控平台；虽然此途径计算的响应时间并不客观，但足以在同一环境下的纵向横向比较，归纳接口整体状况。

-------

# & 二、URL重定向
- 说明：这个功能还是很常用的，平时开发和QA测试都不可避免的要切换环境。特别是多部门合作，服务部署在不同环境上，这绝对可以提高不少效率，通过下面的方法，我们已经实现在客户端可视化的配置Host；

- 大致思路：在恰当的时机，全局替换request：

  ```objc
  + (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
      return [LJDebugURLProtocol p_configureRequest:request];
  }	
  ```

  ​恰当时机，是指**canonicalRequestForRequest**这个函数，替换方式可以写成下面这样：
  ```objc
  + (NSMutableURLRequest *)p_configureRequest:(NSURLRequest *)reqeust {
    NSMutableURLRequest *mutableReqeust = [reqeust mutableCopy];
    
    NSArray <NSData *> *hostModels = [[NSUserDefaults standardUserDefaults] arrayForKey:kLJDebugConfigureHostModels];
    [hostModels enumerateObjectsUsingBlock:^(NSData * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        
        LJDebugConfigureHostModel *hostModel = [NSKeyedUnarchiver unarchiveObjectWithData:obj];
        if (hostModel.selected && hostModel.domain && hostModel.address) {
            
            NSString *absoluteString = [mutableReqeust URL].absoluteString;
            if ([absoluteString rangeOfString:hostModel.domain].location != NSNotFound) {
                
                NSString *newAbsoluteString = [absoluteString stringByReplacingOccurrencesOfString:hostModel.domain withString:hostModel.address];
                mutableReqeust.URL = [NSURL URLWithString:[NSString stringWithFormat:@"%@", newAbsoluteString]]; 
                *stop = YES;
            }
        }
    }];
    
    return mutableReqeust;
}
```
-------

# & 三、模拟请求
- 说明：用过Charles都知道，接口联调环节修改请求的response是件很爽的事儿；有没有想过，这个可以直接在客户端实现…
- 大致思路：在NSURLProtocol的子类中，实现**NSURLConnectionDataDelegate** 的代理方法，直接将respose修改成想要的结果，然后通过**NSURLProtocolClient**抛出去；
不要小看这个功能，发散下去，**过滤请求和返回中的敏感信息**以及**故意制造畸形或非法返回数据来测试程序的鲁棒性** 这些在接口联调环节都可以提高反馈效率；

-------

# & 最后
NSProtocol能做的事情，绝对不仅仅上面提到的这些，相信只要掌握其中的奥义，还有更多可能去实践…

