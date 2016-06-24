---
title: RxSwift快速指南(一)
date: 2016-06-24T19:25:02.000Z
tags:
  - iOS
  - RxSwift
  - Swift
---

# 基本概念

每个Observable队列都仅仅是一个简单的队列，Observable比Swift中的SequenceType优秀在于它能接受异步的元素，这是RxSwift的核心概念。

## Event

队列的语法： `Next* (Error | Completed)?`

Event是一个泛型枚举，一个队列中可以有0个或者多个Next元素，当队列中出现Error和Completed元素时，队列将不再接受Next元素。

```swift
enum Event<Element>  {
    case Next(Element)
    case Error(ErrorType)
    case Completed
}
```

## 创建一个Observable(aka observable sequence)

在Observable+Creation中，RxSwift定义了很多帮助创建Observable的方法，例如`create`, `empty`, `never`, `just`, `error`, `of`, `deferred`, `generate`, `repeatElement`和`using`, 以及对SequenceType和Array的扩展方法`toObservable`。

```swift
func myJust<E>(element: E) -> Observable<E> {
    return Observable.create { observer in
        observer.on(.Next(element))
        observer.on(.Completed)
        return NopDisposable.instance
    }
}
```

这里通过尾随闭包(Trailing Closures)来创建了一个Observable，create方法的定义是：

```swift
public static func create(subscribe: (AnyObserver<E>) -> Disposable) -> Observable<E> {
    return AnonymousObservable(subscribe)
}
```

这里subscribe其实是一个handler，接受一个Observer，返回一个Disposable对象，并作为AnonymousObservable的init参数传入。`AnyObserver<Element>`实现了`ObserverType`协议，我们在该handler内，将`Event`通过`on`发送到队列中去，最后返回的Disposable，Disposable在Observable被`subscribe`后被返回，用作Observable的终止。

## 订阅一个Observable

在Observable被创建后，我们会得到一个Observable，它实现了`ObservableType`协议，在Observable+Extensions中，可以发现一些用于订阅的方法：`subscribe`, `subscribeNext`, `subscribeError`和`subscribeCompleted`。被创建的Observable是不会执行任何代码的，因为它只定义了Observable怎么被创建起来，只有在它被subscribe之后，队列才会真正被创建。

```swift
public func subscribeNext(onNext: (E) -> Void) -> Disposable {
    let observer = AnonymousObserver<E> { e in
      if case .Next(let value) = e {
        onNext(value)
      }
    }
    return self.subscribeSafe(observer)
}
```

## Observable的终止

在一个subscription上调用`dispose`将终止一个被观察的队列，在create时定义的Disposable的销毁代码会被执行。

# 生命周期

在基本概念中，我们接触到了Event，Observable和Observer，他们的生命周期如图所示：

{% asset_img RxSwift.001.png RxSwift life cycle %}

> 本节仅介绍了RxSwift中几个重要的概念和他们是怎么协同工作的，下一节将细化Observable的transform, filter和combine。
