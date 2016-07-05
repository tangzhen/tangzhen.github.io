---
title: RxSwift快速指南(三)
date: 2016-07-03T14:26:33.000Z
tags:
  - iOS
  - RxSwift
  - Swift
---

# Filter

Filter的操作即为过滤，是将Observable队列中不需要的Element去掉，RxSwift为我们提供了例如`debounce \ throttle`, `distinctUntilChanged`, `elementAt`, `filter`, `sample`, `skip`, `take`, `takeLast`和`single`的方法。

## debounce

debounce和throttle其实是一样的，队列中的元素如果和下一个元素的间隔小于了指定的时间间隔，那么这个元素将被过滤掉。

```swift
let times = [
    [ "value": 0, "time": 0.1 ],
    [ "value": 1, "time": 0.4 ],
    [ "value": 2, "time": 0.6 ],
    [ "value": 3, "time": 0.8 ],
    [ "value": 4, "time": 1.0 ],
    [ "value": 5, "time": 1.4 ]
];

times.toObservable()
    .flatMap { item in
        return Observable.of(item["value"]!)
            .delaySubscription(Double(item["time"]!), scheduler: MainScheduler.instance)
    }
    .debounce(0.2, scheduler: MainScheduler.instance)
    .subscribe { print($0) }
```

代码输出结果：

```
Next(0.0)
Next(4.0)
Next(5.0)
Completed
```

## distinctUntilChanged

如果队列中出现相同Element，将被过滤掉。

```swift
let disposeBag = DisposeBag()

Observable.of(1, 2, 3, 3, 3, 4, 5)
    .distinctUntilChanged()
    .subscribe { print($0) }
    .addDisposableTo(disposeBag)
```

## filter

```swift
let disposeBag = DisposeBag()

Observable.of(
    "aa", "ab", "ac",
    "bb", "aa", "cc",
    "ca", "bc", "aa")
    .filter { $0 == "aa" }
    .subscribeNext { print($0) }
    .addDisposableTo(disposeBag)
```

```swift
public func filter(predicate: (E) throws -> Bool) -> Observable<E> {
    return Filter(source: asObservable(), predicate: predicate)
}
```

和其他FP中的filter一样，predicate返回值为布尔值，true则保留这个Element，返回false时这个Element将被过滤掉。

## sample

```swift
let tap = Observable<NSInteger>.interval(0.6, scheduler: SerialDispatchQueueScheduler(internalSerialQueueName: "tap"))

let timerSubscribe = Observable<NSInteger>.interval(0.1, scheduler: SerialDispatchQueueScheduler(internalSerialQueueName: "timer"))
   .sample(tap)
   .subscribeNext({ msecs -> Void in
       print("\(msecs)00ms")
   })

NSThread.sleepForTimeInterval(4)
timerSubscribe.dispose()
```

sample字面意思是简化，通过参数的队列来简化源队列，在官方的Example中没有找到对应例子，官方的描述也是让人想不出来这个方法适用于什么地方，直到看这个[这篇博客](http://rx-marin.com/post/rxswift-rxcocoa-sample-split-laps-timer/)才豁然开朗，需注意的是，参数队列中的元素值将被忽略。

{% asset_img sample.png sample %}

## 其他

其他方法都比较简单，通过方法名都能知道在干什么，官方的playground中也能找到对应的实例，这里就不一一阐述了。
