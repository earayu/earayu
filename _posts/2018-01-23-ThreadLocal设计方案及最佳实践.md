---
title: ThreadLocal设计方案及最佳实践
updated: 2018-01-23 01:15
---

## ThreadLocal设计方案及最佳实践

ThreadLocal的作用是提供线程内的局部变量，API简单易用。不过ThreadLocal的使用不是本文存在的原因，今天研究了一下ThreadLocal的设计思路，想在这里记录一下。



### 问题剖析

为了实现我们的想法`提供线程内的局部变量`，我们需要一个什么样的东西？官方的API大概是这样使用的：

```java
ThreadLocal foo = ThreadLocal.withInitial(()-> 0);
foo.get();
foo.set(1);
foo.remove();
```

显而易见，我们直接操作`ThreadLocal变量`，而变量的值是根据`当前代码执行的上下文（当前线程）`确定的。所以我们起码可以提出2点：

1. 线程和ThreadLocal变量是**多对多**的关系，即一个线程可以有多个ThreadLocal变量；一个ThreadLocal变量也对应多个线程。
2. 为了确定一个值，我们至少需要知道是当前处于**哪个线程**？ 以及是**哪个变量？** 

幸运的是，这2个问题都不难回答。线程当然是**当前线程** ，而变量当然也就是我们**正在操作的变量**。这虽然看起来是2个傻问题，但记住这句话：**这确实是ThreadLocal设计的关键**。



### 方案研究

如果没看过ThreadLocal的源码，只使用过它的API，让我们实现它，我们会怎么做？

（阅读下文时请时刻记住上面提出的2个问题）

（因为个人喜好，使用了python和Java混合代码）

#### 设计1：

 最容易想到的实现方式：当然是为每个变量维护一个线程表，然后根据线程来取得相应的值。

```python
# 设计1的实现
class ThreadLocal:
  	def get():
      	Map<Thread, Object> threadMap = this.threadMap 
        return threadMap.get(Thread.currentThread()).value()
```

这种方式先定位了变量，后定位了线程。值保存在ThreadLocal类的map中。

#### 设计2：

还有一种实现方式：为每个线程维护一个变量表，然后根据变量来取得相应的值。

```python
# 设计2的实现
class ThreadLocal:
 	def get():
 		Map<ThreadLocal, Object> threadLocalMap = Thread.currentThread().threadLocalMap
        return threadLocalMap.get(this).value() # 因为我们操作的是变量，所以this指向当前操作的变量
```

这种方式先定位了线程，后定位了变量。值保存在**Thread类**的map中。



以上2种设计都能满足最初的API，也就是都能满足我们的需求。JDK早期版本使用了设计1的方案，后来改为了设计2的方案。



### 内存泄露

在使用ThreadLocal时，仿佛耳边总有个人在轻声说着：泄露~内存泄露~! 网上的资料大都语焉不详，可能是因为知根知底的人实在太少了。虽然知道解决问题的方法很简单：调用remove()，但总有一种知其然，而不知其所以然的感觉，非常难受。所以花了一天的时间研究ThreadLocal。

![a](/assets/threadlocal.png)

背景知识：Thread类中有一个ThreadLocalMap类型的变量，它没有实现Map接口，但自己实现了类似Map的功能。ThreadLocalMap的数据保存在一个Entry数组中，Entry的key是ThreadLocal，value是实际的值。key有一个特殊的地方在于，它是弱引用的。`弱引用：当所引用的对象在JVM内不再有强引用指向时，GC后weak reference将会被自动回收。` 
基于以上背景知识，我们构造以下场景：

```java
// 位置A： some code here...
ThreadLocal local = new ThreadLocal();  //1
local.set("foo")						//2
local = null;							//3
System.gc();							//4
// 位置B： other code here...
```

以上代码执行到位置B的时候就发生了内存泄露！

第三行的时候我们把local赋值为null，（请看一眼上图的图）可见，对ThreadLocal的引用只剩下Entry的key的弱引用了。紧接着第四行我们手动触发GC，ThreadLocal对象就被回收了，所以Entry的key指向了null。但是这时候Entry对value还是有着强引用的，它不会被回收。更糟糕的是，我们好像没有办法通过**正常渠道**访问到这个value了，这个value此时失去了意义，只是平白无故地占用着我们的空间，直到**该线程结束**。

**以上场景的条件并不苛刻**

以最常见的tomcat + springMVC环境为例：

```java
@RequestMapping(value = "/thread/local")
public String foo() {
    ThreadLocal local = new ThreadLocal();  //1
	local.set("foo")						//2
}
```

以上场景就满足了内存泄露的所有条件：

1. local变量是局部变量，方法结束后对ThreadLocal的强引用消失。下次GC后对ThreadLocal的弱引用也消失、
2. tomcat使用线程池，每次请求取出一个线程，用完之后放回线程池（意味着线程不会结束）

你可以实现一下上面的方法，请求`/thread/local`接口100次，然后在101次的时候打上断点，检查Thread.currentThread().threadLocals，看是否有很多个key为null，而value为"foo"的Entry。



### 最佳实践

1. ThreadLocal在没有线程池使用的情况下，不会存在内存泄露
2. 如果使用了线程池的话，就依赖于线程池的实现，如果线程池不销毁线程的话，那么就会存在内存泄露。所以使用线程池的话，最好在线程工作结束的时候调用一下remove()
3. tomcat等框架使用了线程池，需要调用remove()




PS: 如果在线程池的情况下不调用remove()，除了内存泄露还有另外一个被大家忽略的问题：

线程工作结束，被回收后变量的值没有回收。下次再取出该线程，withInitial()方法将不会执行，所以得到值也是上次的值。这可能会导致程序的逻辑出现错误。

```java
private ThreadLocal<String> threadLocal = ThreadLocal.withInitial(()-> UUID.randomUUID().toString());
```




### 参考资料

[深入分析 ThreadLocal 内存泄漏问题](http://blog.xiaohansong.com/2016/08/06/ThreadLocal-memory-leak/)

[深入理解ThreadLocal的"内存溢出"](http://mahl1990.iteye.com/blog/2347932)

[深入分析 ThreadLocal 内存泄漏问题](http://www.importnew.com/22039.html)

[ThreadLocal可能引起的内存泄露](http://www.cnblogs.com/onlywujun/p/3524675.html)