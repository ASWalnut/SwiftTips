# SwiftTips

译自[vincent-pradeilles](https://github.com/vincent-pradeilles)的[swift-tips](https://github.com/vincent-pradeilles/swift-tips)

# 目录


* [#10 Observing new and old value with RxSwift](#observing-new-and-old-value-with-rxswift)
* [#09 Implicit initialization from literal values](#implicit-initialization-from-literal-values)
* [#08 Achieving systematic validation of data](#achieving-systematic-validation-of-data)
* [#07 Implementing the builder pattern with keypaths](#implementing-the-builder-pattern-with-keypaths)
* [#06 存储函数而不是存储值](#存储函数而不是存储值)
* [#05 定义函数类型上运算符](#定义函数类型上运算符)
* [#04 函数别名](#函数别名)
* [#03 在函数中封装状态](#在函数中封装状态)
* [#02 生成枚举中所有cases](#生成枚举中所有cases)
* [#01 对可选值使用map](#对可选值使用map)

## 存储函数而不是存储值
当一个 `类` 存储值的唯一目的是参数化其函数时（不太懂笔者的参数化其函数是什么意思？），就可以不存储值，而是直接存储函数，这两者在调用位置没有明显的差异。（意思是存储闭包函数，而非存储属性来参数化其函数）

```swift
import Foundation

struct MaxValidator {
    let max: Int
    let strictComparison: Bool
    
    func isValid(_ value: Int) -> Bool {
        return self.strictComparison ? value < self.max : value <= self.max
    }
}

struct MaxValidator2 {
    var isValid: (_ value: Int) -> Bool
    
    init(max: Int, strictComparison: Bool) {
        self.isValid = strictComparison ? { $0 < max } : { $0 <= max }
    }
}

MaxValidator(max: 5, strictComparison: true).isValid(5) // false
MaxValidator2(max: 5, strictComparison: false).isValid(5) // true
```


## 定义函数类型上运算符
函数是Swift中的一级公民类型，因此为它们定义运算符是完全合法的。

```swift
import Foundation

let firstRange = { (0...3).contains($0) }
let secondRange = { (5...6).contains($0) }

func ||(_ lhs: @escaping (Int) -> Bool, _ rhs: @escaping (Int) -> Bool) -> (Int) -> Bool {
    return { value in
        return lhs(value) || rhs(value)
    }
}

(firstRange || secondRange)(2) // true
(firstRange || secondRange)(4) // false
(firstRange || secondRange)(6) // true
```

## 函数别名
Typealiases 非常适合以更全面的方式表达函数签名，这使我们能够轻松定义对其进行操作的函数，从而以一种很好的方式编写和使用一些强大的API。
```swift
import Foundation

typealias RangeSet = (Int) -> Bool

func union(_ left: @escaping RangeSet, _ right: @escaping RangeSet) -> RangeSet {
    return { left($0) || right($0) }
}

let firstRange = { (0...3).contains($0) }
let secondRange = { (5...6).contains($0) }

let unionRange = union(firstRange, secondRange)

unionRange(2) // true
unionRange(4) // false
```

## 在函数中封装状态

通过返回一个闭包来捕获本地变量，可以在函数中封装可变状态（值捕获）

```swift
import Foundation

func counterFactory() -> () -> Int {
    var counter = 0
    
    return {
        counter += 1
        return counter
    }
}

let counter = counterFactory()

counter() // returns 1
counter() // returns 2
```

## 生成枚举中所有cases
> ⚠️ 自从Swift 4.2, `allCases`只需遵循`CaseIterable`协议，就可以在编译时进行合成。下面的实现方式可以放弃了。

```swift
import Foundation

enum MyEnum { case first; case second; case third; case fourth }

protocol EnumCollection: Hashable {
    static var allCases: [Self] { get }
}

extension EnumCollection {
    public static var allCases: [Self] {
        var i = 0
        return Array(AnyIterator {
            let next = withUnsafePointer(to: &i) {
                $0.withMemoryRebound(to: Self.self, capacity: 1) { $0.pointee }
            }
            if next.hashValue != i { return nil }
            i += 1
            return next
        })
    }
}

extension MyEnum: EnumCollection { }

MyEnum.allCases // [.first, .second, .third, .fourth]
```


## 对可选值使用map

if-let是一种安全处理可选值的很好的方式，但是有时候会有点麻烦，下面的cases，使用 `Optional.map()`是一种很好的方法，可以在保持安全性和可读性的同时缩短代码。
```swift
import UIKit

let date: Date? = Date() // or could be nil, doesn't matter
let formatter = DateFormatter()
let label = UILabel()

if let safeDate = date {
    label.text = formatter.string(from: safeDate)
}

label.text = date.map { return formatter.string(from: $0) }

label.text = date.map(formatter.string(from:)) // even shorter, tough less readable
```
