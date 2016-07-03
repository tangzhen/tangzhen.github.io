---
title: RxSwift快速指南(二)
date: 2016-06-27T23:26:10.000Z
tags:
  - iOS
  - RxSwift
  - Swift
---

# Transform

接触过FP的应该都知道map，RxSwift也提供对应的方法将Element进行变形，主要的方法有`buffer`, `flatMap`, `flatMapFirst`, `flatMapLatest`, `map`, `scan`和`window`， 用的比较多的是`map`和`flatMap`。

## map

```swift
let disposeBag = DisposeBag()
Observable.of(1, 2, 3)
    .map { "\($0)\($0)" }
    .subscribeNext { print($0) }
    .addDisposableTo(disposeBag)
```

根据`map`的定义：

```swift
public func map<R>(selector: E throws -> R) -> Observable<R> {
    return self.asObservable().composeMap(selector)
}
```

`selector`这个closure负责将E转换为R，而E这里被定义为Observable.of(1, 2, 3)的Int，每个元素会被转换为String。

## flatMap

```swift
let disposeBag = DisposeBag()
let sequenceInt = Observable.of(1, 2, 3)
let sequenceString = Observable.of("A", "B", "C", "D")

sequenceInt.flatMap { _ in
        return sequenceString
    }
    .subscribe { print($0) }
    .addDisposableTo(disposeBag)
```

`flatMap`的定义：

```swift
public func flatMap<O: ObservableConvertibleType>(selector: (E) throws -> O) -> Observable<O.E> {
    return FlatMap(source: asObservable(), selector: selector)
}
```

来看看Rx官方给出的描述：

map                                                                              | flatMap
-------------------------------------------------------------------------------- | -----------------------------------------------------------------------------------------------------------------------------
Transform the items emitted by an Observable by applying a function to each item | Transform the items emitted by an Observable into Observables, then flatten the emissions from those into a single Observable

`map`和`flatMap`的区别在于`map`的selector将用于Observable中的每一个Element，操作完成后，Element的数量和源Observable中的是一样的，而`flatMap`的selector的调用次数和源Element数量一致，但是由于selector的返回值为一个Observable，所以在操作完成后，得到的Elements为源Element数量*selector返回的Observable的Element数量，然后将所有Element放入一个Observable中，类似于`[[1, 2], [2, 3], [3, 4]]` -> `[1, 2, 2, 3, 3, 4]`。

{% asset_img flatMap.png flatMap %}

## flatMapLatest

比起`map`和`flatMap`，要稍微难理解一点：

```swift
let disposeBag = DisposeBag()

[1, 2, 3].toObservable()
    .flatMapLatest { value  in
        return ["\(value)a", "\(value)b"].toObservable()
    }
    .subscribe { print($0) }
    .addDisposableTo(disposeBag)
```

根据[RxJS flatMapLatest](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/operators/flatmaplatest.md)这个代码期望的结果是（RxSwift和Rx系的其他语句具有相似接口，其中RxJava对应的方法是switchMap，在[RxJS Source Code](https://github.com/Reactive-Extensions/RxJS/blob/master/src/core/perf/operators/flatmaplatest.js)也能发现其实它和switchMap是一个函数）：

```bash
Next(1a)
Next(2a)
Next(3a)
Next(3b)
Completed
```

但是实际上得到的输出结果是：

```bash
Next(1a)
Next(1b)
Next(2a)
Next(2b)
Next(3a)
Next(3b)
Completed
```

出现这种情况的原因在github的[Get Start](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/GettingStarted.md#creating-your-own-observable-aka-observable-sequence)有解释，问题出现在selector中的`toObservable`，它的实现使用了`Sequence`:

```swift
public func toObservable(scheduler: ImmediateSchedulerType? = nil) -> Observable<Generator.Element> {
    return Sequence(elements: self, scheduler: scheduler)
}
```

`Sequence`其实是一个同步队列的实现，在`subscribe`调用前会生成Next和Completed，因为无论哪种disposable被返回，生成elements的过程都不能被打断。在这种情况下，`flatMapLatest`和`flatMap`的输出结果一样。

期望的版本:

```swift
let disposeBag = DisposeBag()
var charValues: [Variable<Character>] = [Variable("a"), Variable("a"), Variable("a")]

[0, 1, 2].toObservable()
    .flatMapLatest { value -> Observable<Character> in
        print("Int value: \(value)")
        return charValues[value].asObservable()
    }
    .subscribe { print($0) }
    .addDisposableTo(disposeBag)

charValues[2].value = "b"
charValues[0].value = "c"   // nothing happen
charValues[1].value = "d"   // nothing happen
```

{% asset_img flatMapLatest.png flatMapLatest %}

和`flatMap`一样，`flatMapLatest`也会新生成一个Observable的队列，但不同的是它不会合并所有的新Observable中的Element，它会switch到最后一个Observable上（`switchMap`这个名字感觉更容易让人理解一点），先前建立的Observable将不再被监听，所以代码中charValues只有最后一个Observable还在被subscribe。
