---
title: Swift-属性包装器
date: 2022-02-10 16:48:14
categories:
- iOS
tags:
- 属性包装器
keywords:
- Swift,属性包装器
description:
images:
---

### Property wrapper
Property wrapper 类似于 java 中的注解，一种给属性附加逻辑的特性。在类、结构体、枚举声明时使用该特性，可以让其成为一个属性包装器。属性包装器可复用统一规则对属性进行包装，降低重复工作。
<!-- more -->
```swift
@propertyWrapper struct increase { }
```

使用 @propertyWrapper 声明后，必须定义一个 wrappedValue 属性和 `init(wrappedValue: Int) ` 构造器方法。

wrappedValue 属性是一个计算类型属性，包装处理在 wrappedValue 的 `set`、`get` 方法中进行，由于计算类型属性无法储值，因此还需另定义一个属性存值

`init(wrappedValue: Int) ` 在包装器对属性声明时会自动调用，根据属性默认值进行初始化。

```swift
@propertyWrapper struct increase {
    /// 存储值
    private var truelyValue: Int = 0
    /// 包装
    var wrappedValue: Int {
        get {
            truelyValue + 20
        }
        set {
            truelyValue = newValue
        }
    }
    
    init(wrappedValue: Int) {
        self.wrappedValue = wrappedValue
    }
}

使用：
class Test {
    @increase var a: Int = 0
}
```

在修饰属性时，属性包装器的 wrappedValue 的入参是可选的，如上面不显示传入 wrappedValue 就需要给予属性初始值。若显示传入 wrappedValue，不能给予属性初始值。
```swift
class Test {
    @increase(wrappedValue: 6)
    var a: Int
}
```

`init(wrappedValue: Int)` 构造器可以追加初始化参数，
```swift
@propertyWrapper struct increase {
    ....
    var maxValue: Int
    
    init(wrappedValue: Int, maxValue: Int) {
        self.wrappedValue = wrappedValue
        self.maxValue = maxValue
    }
}

class Test {
    @increase(wrappedValue: 6, maxValue: 100)
    var a: Int
}
```
### 组合包装器
可以对一个属性声明多个属性包装器，执行顺序为从右到左。
```swift
@propertyWrapper struct increase: CustomStringConvertible {
    /// 存储值
    private var truelyValue: Int = 0
    /// 包装
    var wrappedValue: Int {
        get {
            truelyValue + 20
        }
        set {
            truelyValue = newValue
        }
    }
    
    init(wrappedValue: Int) {
        self.wrappedValue = wrappedValue
    }
    
    var description: String {
        return "\(wrappedValue)"
    }
    
}

@propertyWrapper struct resultPrint<Value> {
    /// 包装
    var wrappedValue: Value {
        didSet {
            print("\(wrappedValue)")
        }
    }
    
    init(wrappedValue: Value) {
        self.wrappedValue = wrappedValue
    }
    
}


struct Test {
    @resultPrint @increase var a: Int = 5
}


var t = Test.init()
t.a = 15
output：35
```
在上面到示例中，首先执行的是 @increase，然后是 @resultPrint，@resultPrint 会接收到 @increase 包装器对象作为其 wrappedValue 值。

