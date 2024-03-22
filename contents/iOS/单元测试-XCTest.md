---
title: 单元测试-XCTest
date: 2021-12-09 15:23:38
categories:
- iOS
tags:
- 单元测试
keywords:
- 单元测试,XCTest
description:
images:
---
> 单元测试（Unit Testing）又称为模块测试，是针对程序模块软件设计来进行正确性检验的测试工作。程序单元是应用的最小可测试部件。对于面向对象编程，最小单元就是方法，包括基类、抽象类、或者派生类中的方法。

> 单元测试通常由软件开发人员编写，用于确保他们所写的代码符合软件需求和遵循开发目标。通常来说，每修改一次程序就会进行最少一次单元测试，在编写程序的过程中前后很可能要进行多次单元测试，以证实程序达到工作目标要求。
<!-- more -->
### Unit Tests
![image](https://destinmure.github.io/pic/postImage/单元测试-XCTest/psb-01.png)
创建项目或文件时，底部勾选 Unit Tests 会创建对应的测试类，该类继承 XCTestCase，提供测试方法如下：
- setUpWithError：在进行测试前会调用该方法，用于准备测试中需要的条件如：对象初始化、测试数据准备等等。
- tearDownWithError：在测试结束后会调用该方法
- testExample：功能测试用例的示例，可自定义创建多个类似的方法，方法命名规则：test+XXX，且方法不能声明传入参数，测试方法的执行顺序是按照方法名中 test 后面的字符顺序来执行的。
- testPerformanceExample：性能测试用例的示例，可自定义创建，命名规则和 testExample 一样，他们区别在于，testPerformanceExample 中需调用 measure，将需要性能测试的代码放入 measure的 block 参数中，measure的会执行多次，运行时间比对设定的标准值和偏差判断是否可以通过测试。

测试方法前面有菱形按钮，点击该按钮可单独执行该测试方法，也可以使用快捷键 command + u 运行整个测试单元，正确运行后显示绿色对勾，运行错误会显示红色叉号。

### XCTAssert-断言
大部分的测试方法使用断言决定的测试结果，所有断言都有一个类似的形式：比较，表达式为真假，强行失败等。断言方法如下：
```Objective-C
public func XCTAssert(_ expression: @autoclosure () throws -> Bool, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line)
public func XCTAssertEqual<T>(_ expression1: @autoclosure () throws -> T, _ expression2: @autoclosure () throws -> T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line) where T : Equatable
public func XCTAssertEqual<T>(_ expression1: @autoclosure () throws -> T, _ expression2: @autoclosure () throws -> T, accuracy: T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line) where T : FloatingPoint
public func XCTAssertEqual<T>(_ expression1: @autoclosure () throws -> T, _ expression2: @autoclosure () throws -> T, accuracy: T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line) where T : Numeric
public func XCTAssertFalse(_ expression: @autoclosure () throws -> Bool, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line)
public func XCTAssertGreaterThan<T>(_ expression1: @autoclosure () throws -> T, _ expression2: @autoclosure () throws -> T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line) where T : Comparable
public func XCTAssertGreaterThanOrEqual<T>(_ expression1: @autoclosure () throws -> T, _ expression2: @autoclosure () throws -> T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line) where T : Comparable
public func XCTAssertIdentical(_ expression1: @autoclosure () throws -> AnyObject?, _ expression2: @autoclosure () throws -> AnyObject?, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line)
public func XCTAssertLessThan<T>(_ expression1: @autoclosure () throws -> T, _ expression2: @autoclosure () throws -> T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line) where T : Comparable
public func XCTAssertLessThanOrEqual<T>(_ expression1: @autoclosure () throws -> T, _ expression2: @autoclosure () throws -> T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line) where T : Comparable
public func XCTAssertNil(_ expression: @autoclosure () throws -> Any?, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line)
public func XCTAssertNoThrow<T>(_ expression: @autoclosure () throws -> T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line)
public func XCTAssertNotEqual<T>(_ expression1: @autoclosure () throws -> T, _ expression2: @autoclosure () throws -> T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line) where T : Equatable
public func XCTAssertNotEqual<T>(_ expression1: @autoclosure () throws -> T, _ expression2: @autoclosure () throws -> T, accuracy: T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line) where T : FloatingPoint
public func XCTAssertNotEqual<T>(_ expression1: @autoclosure () throws -> T, _ expression2: @autoclosure () throws -> T, accuracy: T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line) where T : Numeric
public func XCTAssertNotIdentical(_ expression1: @autoclosure () throws -> AnyObject?, _ expression2: @autoclosure () throws -> AnyObject?, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line)
public func XCTAssertNotNil(_ expression: @autoclosure () throws -> Any?, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line)
public func XCTAssertThrowsError<T>(_ expression: @autoclosure () throws -> T, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line, _ errorHandler: (_ error: Error) -> Void = { _ in })
public func XCTAssertTrue(_ expression: @autoclosure () throws -> Bool, _ message: @autoclosure () -> String = "", file: StaticString = #filePath, line: UInt = #line)
public func XCTExpectFailure(_ failureReason: String? = nil, options: XCTExpectedFailure.Options = .init())
public func XCTExpectFailure<R>(_ failureReason: String? = nil, options: XCTExpectedFailure.Options = .init(), failingBlock: () throws -> R) rethrows -> R
public func XCTFail(_ message: String = "", file: StaticString = #filePath, line: UInt = #line)
```
### 异步逻辑测试

测试异步方法时，例如网络请求等耗时操作，由于执行结果不是立即就能获取到，XCTest 提供了一些辅助方法，如下例所示：
```swift
let expectation = XCTestExpectation()
request { // 请求回调
    expectation.fulfill()
}

// 等待 expectation 执行fulfill，2 秒后超时
waitForExpectations(timeout: 2) { eror in
  // 超时
}
```
waitForExpectationsWithTimeout 方法会在规定时间内，等待 XCTestExpectation 执行 fulfill()，规定时间内不执行就会执行超时闭包。

此外还可配合以下方法实现对异步操作的测试：
```Objective-C
// 根据 NSPredicate 判读对象的属性
open func expectation(for predicate: NSPredicate, evaluatedWith object: Any?, handler: XCTNSPredicateExpectation.Handler? = nil) -> XCTestExpectation

// 根据 notificationName 通知，监听是否收到通知
open func expectation(forNotification notificationName: NSNotification.Name, object objectToObserve: Any?, notificationCenter: NotificationCenter, handler: XCTNSNotificationExpectation.Handler? = nil) -> XCTestExpectation
```
使用示例如下：
```swift
expectation(for: NSPredicate.init(format: "name == 'car'"), evaluatedWith: car) {
   // 判断通过
}
// 等待 expectation 执行fulfill，2 秒后超时
waitForExpectations(timeout: 2) { eror in
  // 超时
}


expectation(forNotification: NSNotification.Name(rawValue: "NSNotification"), object: nil) { notification in
   // 已接收通知      
}
// 等待 expectation 执行fulfill，2 秒后超时
waitForExpectations(timeout: 2) { eror in
  // 通知超时
}
```


### UITest
UITest 用于测试 UI 交互逻辑，对应的测试类与 Unit Test一致
```swift
class LearnXCTestUITests: XCTestCase {

    override func setUpWithError() throws {
        // Put setup code here. This method is called before the invocation of each test method in the class.

        // In UI tests it is usually best to stop immediately when a failure occurs.
        continueAfterFailure = false

        // In UI tests it’s important to set the initial state - such as interface orientation - required for your tests before they run. The setUp method is a good place to do this.
    }

    override func tearDownWithError() throws {
        // Put teardown code here. This method is called after the invocation of each test method in the class.
    }

    func testExample() throws {
        // UI tests must launch the application that they test.
        
        let app = XCUIApplication()
        app.launch()

        // Use recording to get started writing UI tests.
        // Use XCTAssert and related functions to verify your tests produce the correct results.
    }

    func testLaunchPerformance() throws {
        if #available(macOS 10.15, iOS 13.0, tvOS 13.0, watchOS 7.0, *) {
            // This measures how long it takes to launch your application.
            measure(metrics: [XCTApplicationLaunchMetric()]) {
                //XCUIApplication().launch()
            }
        }
    }
}
```
为方便编写 UI 测试，Xcode 提供 UI 行为录制，光标定位到 testExample 方法中，点击底部红点开启录制，当点击 UI 时会自动生成 UI 操作代码。
![image](https://destinmure.github.io/pic/postImage/单元测试-XCTest/psb-02.png)
