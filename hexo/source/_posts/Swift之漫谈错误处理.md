---
title: Swift之漫谈错误处理
date: 2017-02-01 19:57:19
tags:
categories: Swift
---


> 一个人知道了自己的短处，能够改过自新，就是幸福的。——莎士比亚

-------

# & 引子
代码遇到意外，在所难免，Swift 发展到此时的 3.0 版本，对错误处理机制不断完善，面对不同场景，现在我们已经可以有多种选择。风格依旧，本文依然不是如何进行错误处理的严肃科普文，有这个需求的同学，Google 或者官方文档更适合你。下面，我们就聊聊**错误处理**在 Swift 这门语言中玩出了什么新花样儿…

-------
# & 错误与异常
先来看下 Objective-C，错误和异常有着明显的区别，这体现在设计之初的本意上。**错误**的范畴大多会归咎于正常情况，只是结果并不符合我们的预期，例如文件读写失败，产生错误的原因可能是磁盘空间不足等等，网络请求失败，产生错误的原因就更复杂了；但是**异常**大多是我们的代码漏洞造成的，对应的处理类是 NSException。
我个人并不喜欢二者这种刻意而且明显区别。特别是针对错误，下面这样的例子应该很常见：

```objc
NSError *error = nil;
BOOL success = [[NSFileManager defaultManager] moveItemAtPath:@"/path/to/target"
                                                       toPath:@"/path/to/destination"
                                                        error:&error];
if (!success) {
    NSLog(@"%@", error);
}
// 更常见的写法
[[NSFileManager defaultManager] moveItemAtPath:@"/path/to/target"
                                                       toPath:@"/path/to/destination"
                                                        error:nil];
```
是的，正如代码中更常见的写法，我们很容易偷懒绕开错误处理，这明显违背了 API 的设计初衷，也影响了代码的鲁棒性并给后续问题排查带来不便。这在 Swift 中得到了有效改善。<!-- more -->

-------

# & throws
这是个让我感到惊讶的特性！从此，我们可以统一错误和异常的处理，在一套代码逻辑中完成非正常情况的处理。
```swift
func canThrowErrors() throws -> String
func cannotThrowErrors() -> String
```
像上面那样，一个标有 throws 关键字的函数被称作 throwing 函数。throwing 函数可以在其内部抛出错误，并将错误传递到函数被调用时的作用域。然后在函数被调用处，可以这样处理错误：
```swift
do {
	try expression
	statements
} catch pattern 1 {
	statements
} catch pattern 2 where condition {
statements }
```
如上面所示，在 do-catch 代码块儿中，我们可以捕获函数抛出的错误，但是这里有个明显的事实，这个事实也被业界激烈讨论，褒贬不一。那就是throws抛出的错误是无类型的！我们先试着从合理的角度解释这种设计：无类型错误确实可以让我们的类型签名保持简单，比如要是我们需要指定错误 类型时，多余一种错误会发生的情况下，类型签名很快就会变得更加复杂。但是，这么设计的不足也显而易⻅:没有机制可以指定只有一种错误会发生，这导致了额外的模板代码。此外，throws 只能与函数协同工作，而不能作用于其他类型，这为我们使用它增添了不必要的复杂度。乐观的情况是，蹩脚的情况并不多见，在大多数场景中，throws 仍然是我们可以信赖的选择。

-------

# & Result 类型
 上面提到的 throws 是作用于函数的，我们能否自定义一种作用于类型的错误表现形式，在 Swift 中可选值其实就是有两个成员的枚举:一个不包含关联值的 .None 或者 nil，还有一个关联值 Some。Result 跟可选值很相似，也是两个成员组成的枚举，先来看下Result的泛型枚举结构： 
```swift
enum Result<A> {
	case Failure(ErrorType)
	case Success(A) 
}
```
显然，枚举中错误用符合 ErrorType 协议的类型的值来表示。然后，就可以像下面这样处理 Result：

```swift
enum FileError: ErrorType { 
	case FileDoesNotExist 
	case NoPermission
}

func contentsOfFile( lename: String) -> Result<String>

let result = contentsOfFile("input.txt") switch result {
	case let .Success(contents): print ( contents)
	caselet .Failure(error):
	if let  leError = error as? FileError
	}
	where  leError == .FileDoesNotExist {
	print("Empty  le") } else {
	// 处理错误 
	}
}
let result = contentsOfFile("input.txt") switch result {
	case let .Success(contents): print ( contents)
	caselet .Failure(error):
	if let  leError = error as? FileError
	}
	where  leError == .FileDoesNotExist {
	print("Empty  le") } else {
	// 处理错误 
	}
}
```
这给了我们自定义错误类型的可能。Swift 内建的错误处理的实现方式和这很类似，只不过使用了不同的语法。Swift 没有使用返回 Result 的方式来表示失败，而是将方法标记为上面提到的 throws。

-------
# & 其他可能
在Swift的错误处理机制中，我们还有很多选择，下面简单列举几种可能：
* fatalError
在遇到确实因为输入的错误无法使程序继续运行的时候，我们一般考虑以产生致命错误 (fatalError) 的方式来终止程序。
```Swift
//  @noreturn，表示调用这个方法可以不再需要返回值
@noreturn func fatalError(@autoclosure message: () -> String = default,
                                          file: StaticString = default,
                                          line: UInt = default)
```
* 断言
在 Debug 编译环境下，输入参数的的条件检查，这是常用的手段。
    ```swift
    func assert(@autoclosure condition: () -> Bool,
                @autoclosure _ message: () -> String = default,
    			file: StaticString = default,
    			line: UInt = default)
    ```
* 可选值
在处理有可能是 nil 的值时，可选值会非常有用。特别是当我们对错误类型不感兴趣，或者只有一种错误时，我们使用可选值更清晰简洁。

-------

# & 最后
正如上面讨论的，Swift 给我们处理错误提供了多种选择，但是如何在特定场景选择恰当的错误处理机制，需要我们不断揣摩体会。当然，Swift 还很年轻，相信上面提到的问题，在语言的不断发展迭代中会得到优化，我们充满期待！

-------
扩展阅读
[FATALERROR](http://swifter.tips/fatalerror/)
[断言](http://swifter.tips/assert/)

