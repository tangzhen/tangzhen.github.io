---
title: RxSwift快速指南(四)
date: 2016-07-05T21:59:00.000Z
tags:
  - iOS
  - RxSwift
  - Swift
---

# Combine

有时数据源并不只一个，比如用户登录，我们需要将用户名和密码惊喜合并验证，这时我们可以使用RxSwift为我们提供的合并，`combineLatest`, `merge`, `startWith`, `switchLatest`和`zip`等。

## combineLatest

```swift
example("combineLatest") {
let disposeBag = DisposeBag()

let stringSubject = PublishSubject<String>()
let intSubject = PublishSubject<Int>()

Observable.combineLatest(stringSubject, intSubject) { stringElement, intElement in
    "\(stringElement) \(intElement)"
    }
    .subscribeNext { print($0) }
    .addDisposableTo(disposeBag)

stringSubject.onNext("A")
stringSubject.onNext("B")
intSubject.onNext(1)
intSubject.onNext(2)
stringSubject.onNext("AB")
intSubject.onNext(3)
}
```

`combineLatest`和`zip`比较像，通过一个resultSelector的closure，将不同队列中的Element进行合并，最终队列Element的数量是当所有队列都有值后，这些队列在出现的Element数量加一。

## merge

```swift
let disposeBag = DisposeBag()

let subject1 = PublishSubject<String>()
let subject2 = PublishSubject<String>()

Observable.of(subject1, subject2)
    .merge()
    .subscribeNext { print($0) }
    .addDisposableTo(disposeBag)

subject1.onNext("A")
subject1.onNext("B")
subject2.onNext("1")
subject2.onNext("2")
subject1.onNext("AB")
subject2.onNext("3")
```

`merge`将所有队列中的元素按照出现的顺序合并到一个队列中去。

## startWith

`startWith`用于在一个队列前加入一些Elements。

```swift
let disposeBag = DisposeBag()

Observable.of(1, 2, 3, 4)
    .startWith("A")
    .startWith("B", "C", "D")
    .subscribeNext { print($0) }
    .addDisposableTo(disposeBag)
```

```swift
public func startWith(elements: E ...) -> Observable<E> {
   return StartWith(source: self.asObservable(), elements: elements)
}
```

`elements`这里是一个可变参数，可接受一个或者多个元素，需要注意的是如果在Observable中加入debug信息，后加入的Element并不在Next Event里，这可能会给你的调试带来一些麻烦，可能这是一个Debug和Producer在一起的bug。

```bash
--- startWith example ---
B
C
D
A
2016-07-06 19:40:31.382: startWith -> subscribed
2016-07-06 19:40:31.383: startWith -> Event Next(1)
1
2016-07-06 19:40:31.383: startWith -> Event Next(2)
2
2016-07-06 19:40:31.383: startWith -> Event Next(3)
3
2016-07-06 19:40:31.384: startWith -> Event Next(4)
4
2016-07-06 19:40:31.384: startWith -> Event Completed
2016-07-06 19:40:31.384: startWith -> disposed
```

## switchLatest

```swift
let disposeBag = DisposeBag()

Observable.range(start: 0, count: 3)
    .map { Observable.range(start: $0, count: 3) }
    .switchLatest()
    .subscribeNext { print($0) }
    .addDisposableTo(disposeBag)
```

map后再switchLatest其实和flatMapLatest的结果是一样的，需要理解的是switch队列以后，以前队列的Element将不在发送。

{% asset_img switchLatest.png switchLatest %}

## zip

```swift
let disposeBag = DisposeBag()

let range = Observable.range(start: 0, count: 5);
Observable.zip(range, range.skip(1), range.skip(2)) { "\($0):\($1):\($2)" }
    .subscribeNext { print($0) }
    .addDisposableTo(disposeBag)
```

`zip`可以和`combineLatest`对比理解，zip每次在每个队列中取一个元素进行处理，元素在对应的队列中index是一样，产生的Element数量和最少的队列Element数量一样。
