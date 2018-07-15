---
title: ReactiveX
date: 2018-06-05 22:40:50
tags:
---

**（翻译 [ReactiveX 官网](http://reactivex.io/intro.html)）**

[ReactiveX](http://reactivex.io/intro.html) 是通过观察者序列实现的“异步”和“基于事件”的开发范式的类库。

基于[观察者模式](http://en.wikipedia.org/wiki/Observer_pattern)，支持一系列的“数据” and/or “事件” 并且基于此可以添加一系列的“操作”；基于此类抽象可以实现：更少的线程、异步、线程安全、并发的数据结构、NIO。

观察者模式提供了一种完美的解决方案，来填补多项异步序列间的异步序列接入。

（翻译不出来：Observables fill the gap by being the ideal way to access asynchronous sequences of multiple items）

|              | single items        | multiple items          |
| ------------ | ------------------- | ----------------------- |
| synchronous  | T getData()         | Iterable<T> getData()   |
| asynchronous | Future<T> getData() | Observable<T> getData() |

有时候这个被称为“[函数式响应式编程](https://en.wikipedia.org/wiki/Functional_reactive_programming)”，但是这是不合适的叫法。ReactiveX也许是函数式的，也许是响应式的，但是“函数式响应式编程”是另一码事情。一个主要的不同点是：函数式响应式编程基于“值”来操作，“值”是随着时间不停的变化的。而ReactiveX基于“非连续”的“值”， 值是随着时间被发出（emitted）。（参考： [Conal Elliott’s work for more-precise information on functional reactive programming](https://github.com/conal/essence-and-origins-of-frp)）



## 为什么使用观察者模式

ReactiveX 使使用者通过定义一个集合来定义一系列的“简单的、可组合的操作”，来处理一系列的异步事件；他使混乱的web回调变得简单，进而使代码更具有可读性，更少的bug。



### 可组合

[Java Futures](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Future.html) 技术是最快捷的方式来实现 [a single level of asynchronous execution](https://gist.github.com/4670979) 但是，当存在嵌套调用场景时futures技术会造成[non-trivial complexity](https://gist.github.com/4671081) 。

很难[使用future完美的组合条件式的异步处理流程](https://gist.github.com/4671081#file-futuresb-java-L163)（或者说是不可能实现的，因为每一个运行时的请求的延迟是变化的）。这当然[可以做到](https://www.amazon.com/gp/product/0321349601?ie=UTF8&tag=none0b69&linkCode=as2&camp=1789&creative=9325&creativeASIN=0321349601)， 但是会迅速的变得复杂（进而易错）或者过早的阻塞在`Future.get()`, 这样会削减异步带来的收益。

ReactiveX Observables， 在另一方面，倾向于[组合流程并顺序化异步数据](https://github.com/Netflix/RxJava/wiki/How-To-Use#composition)。



### 灵活

ReactiveX Observables 不仅仅支持emission单个scalar values（就像futures那样），并且支持一个序列的值甚至无限的流。`Observable` 是可以处理这些场景的一个抽象。Observable具备迭代器一样的灵活和优雅。

Observable 是异步的推模型 [“dual”](http://en.wikipedia.org/wiki/Dual_(category_theory)) 对比与同步模式下迭代器的拉模型

| event                | Iterable (pull)    | Observable (push)  |
| -------------------- | ------------------ | ------------------ |
| retrieve data（查询数据）  | T next()           | onNext(T)          |
| discover error（发现错误） | throws `Exception` | onError(Exception) |
| complete（完成）         | !hasNext()         | onCompleted()      |



### 不局限

（译者注：对于被监听方来说）

ReactiveX 不倾向于任何特定的并发或异步实现方式。Observable可以通过线程池、event loops、nio、actors（比如akka）实现，或者其他任何实基于你的经验的现方式。调用方把所有的和Observable相关的代码认为是异步的，无论底层的实现是阻塞的或非阻塞的也无论是什么实现方式。

Observable 如何实现?

```java
public Observable<data> getData();
```

- does it work synchronously on the same thread as the caller?
- does it work asynchronously on a distinct thread?
- does it divide its work over multiple threads that may return data to the caller in any order?
- does it use an Actor (or multiple Actors) instead of a thread pool?
- does it use NIO with an event-loop to do asynchronous network access?
- does it use an event-loop to separate the work thread from the callback thread?

从监听者的视角来看，这都无所谓。

提一点：基于ReactiveX你可以随时改变你的主意，彻底的改变Observable的实现方式，不需要改变调用方的代码。



### 回调有自身的问题

`回调`可以解决过早的阻塞在`Future.get()`的问题，它原生的高效，因为他们在response ready的时候执行。但是和futures一样，回调比较适用于单层的异步执行，[嵌套时会变得不好用](https://gist.github.com/4677544)。



### ReactiveX有多语言实现

（吹牛逼的内容不翻译了。）



## 响应式编程

ReactiveX 提供了[一组操作器](http://reactivex.io/documentation/operators.html)，使用者可以过滤、选择、转换、组装、组合Observables。这样就支持了高效的执行和调度。

可以把Observable当做一个“推”，类比于迭代器的“拉”。作为一个[迭代器](http://docs.oracle.com/javase/7/docs/api/java/lang/Iterable.html)，消费者从生产者拉取值，当前线程会被阻塞直到值到达。相形之下，Observables生产值并把它推送给消费者。这种方式更加灵活，因为值可以同步或异步的到达。

下面的样例代码，展示迭代器和Observable在高阶函数方面的相似之处：

**Iterable**

```java
getDataFromLocalMemory()
  .skip(10)
  .take(5)
  .map({ s -> return s + " transformed" })
  .forEach({ println "next => " + it })
```

**Observable**

```java
getDataFromNetwork()
  .skip(10)
  .take(5)
  .map({ s -> return s + " transformed" })
  .subscribe({ println "onNext => " + it })
```

Observable 添加了两个遗漏的语义（相对于GOF Observer pattern）：

- 生产者通知消费者不再有数据的能力（`onCompleted` method）
- 生产者通知消费之发生错误的能力（`onError` method）

基于这些添加项，ReactiveX调和了Iterable 和 Observable 两种模式， 他们唯一的区别是数据流的方向。这非常重要，因为任何可以在Iterable上执行的动作，用户都可以在Observable来做。