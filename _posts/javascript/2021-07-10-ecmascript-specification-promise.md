---
layout: post
title:  "ECMAScript规范 Promise部分翻译"
date:   2021-07-10 23:10:05 +0800
categories: javascript
tags: ecma
---
# 27.2 Promise Object

Promise就是一个对象，作为延迟计算或者异步的计算的最终结果的占位者
任何Promise对象处于三种互相排斥的状态中的一种：fullfilled，rejected和pending：

- p是一个状态为fullfilled的promise，p.then(f, r)会马上将一个执行函数f的job加入队列
- p是一个状态为rejected的promise，p.then(f, r)会马上将一个执行函数r的job加入队列
- 一个promise的状态既不是fulfilled也不是rejected的时候，状态是pending

一个promise在不是pending的状态时被称为已经settled，比如在fulfilled或者rejected的状态
一个promise是resolved在它settled的时候或者因为匹配另一个promise的状态而被"locked in"。尝试修改一个settled的promise是不会修改promise的状态的。一个未resolved的promise是unresolved。一个unresolved的promise总是pending状态。一个resolved的promise也许是pending，fulfilled或者rejected

## 27.2.1 Promise Abstract Operations

### 27.2.1.1 PromiseCapability Records

一个PromiseCapability Record是一个用于概述一个promise对象和可以用于resolving或rejecting这个promise的方法的Record。PromiseCapability Record由NewPromiseCapability这个抽象操作生成

PromiseCapalitity Records包含的所有内容列在Table 70

Table 70:PromiseCapability Record Fields

| 名字        | 值           | 含义                             |
| ----------- | ------------ | -------------------------------- |
| [[Promise]] | 一个对象     | 一个promise对象                  |
| [[Resolve]] | 一个函数对象 | 一个用于resolve给定promise的函数 |
| [[Reject]]  | 一个函数对象 | 一个用于reject给定promise的函数  |

### 27.2.1.2 PromiseReaction Records

PromiseReaction是用于存放一个promise在用给定值resolved或者rejected后如何运行的信息。PromiseReaction记录被PerformPromiseThen抽象操作生成，被Abstract Closure使用，被NewPromiseReactionJob返回
PromiseReaction Records包含的所有内容列在Table 71
Table 71:PromiseReaction Record Fields

| 名字           | 值                                          | 含义                                                         |
| -------------- | ------------------------------------------- | ------------------------------------------------------------ |
| [[Capability]] | 一个PromiseCapability Record，或者undefined | 这个record提供reaction handler给的promise的能力记录          |
| [[Type]]       | Fulfill或Reject                             | 当[[Handler]]是empty时用于确定promise状态的类型              |
| [[Handler]]    | 一个JobCallback Record，或者empty           | 使用传入的值运行的函数，返回的值将决定提供的promise的状态如何被确定。如果[[Handler]]是empty，[[Type]]的值对应的函数将被执行 |

### 27.2.1.3 CreateResolvingFunction(*promise*)

#### 27.2.1.3.1 Promise Reject Functions

一个promise reject函数就是拥有[[Promise]]和[[AlreadyResolved]]内部插槽的内建函数。

当一个promise reject函数，被传入参数reason并运行时，会执行下面的几步

    1. F是active function object
    2. 判断F是否存在值是Object类型的[[Promise]]内部插槽
    3. promise是F.[[Promise]]
    4. alreadyResolved是F.[[AlreadyResolved]]
    5. 如果alreadyResolved.[[Value]]是true，返回undefined
    6. 设置alreadyResolved.[[Value]]为true
    7. 返回RejectPromise(promise, reason)

promise reject 函数的length属性值是1

#### 27.2.1.3.2 Promise Resolve Functions

一个promise resolve函数就是拥有[[Promise]]和[[AlreadyResolved]]内部插槽的内建函数。

当一个promise resolve函数，被传入参数resolution并运行时，会执行下面的几步

    1. F是active function object
    2. 判断F是否存在值是Object类型的[[Promise]]内部插槽
    3. promise是F.[[Promise]]
    4. alreadyResolved是F.[[AlreadyResolved]]
    5. 如果alreadyResolved.[[Value]]是true，返回undefined
    6. 设置alreadyResolved.[[Value]]为true
    7. 使用SameValue(resolution, promise)判断resolution和promise是否一样，如果一样，则
       1. 新创建一个TypeError对象，命名为selfResolutionError
       2. 返回RejectPromise(promise, selfResolutionError)
    8. 使用Type(resolution)判断resolution的类型不是一个对象，则
       1. 返回FulfillPromise(promise, resolution)
    9. 使用Get(resolution, "then")获取resolution的then属性
    10. 如果then是一个abrupt completion，则
            1. 返回 RejectPromise(promise, then.[[Value]])
    11. thenAction是then.[[Value]]
    12. 使用IsCallable(thenAction)判断thenAction是否可以执行，如果是false，则
            1. 返回FulfillPromise(promise, resolution)
    13. 使用HostMakeJobCallback(thenAction)生成thenJobCallback
    14. 使用NewPromiseResolveThenableJob(promise, resolution, thenIobCallback)生成job
    15. 执行HostEnqueuePromiseJob(job.[[Job]], job.[[Realm]]),将job加入微任务队列
    16. 返回undefined

promise resolve 函数的length属性值是1

## 27.2.2 Promise Job

### 27.2.2.1 NewPromiseReactionJob(reaction, argument)

抽象操作NewPromiseReactionJob需要参数reaction和argument。这返回一个Job Abstract Closure（微任务闭包），任务会执行使用合适的处理函数处理传来的参数（reaction参数里有handler函数），并用处理函数的返回值去resolve或reject相关的promise。当被调用时会执行以下步骤：

  1. 声明job，这是一个新的Job Abstract Closure，没有任何参数，记录着reaction和argument，当job被被执行时会走下方的步骤：
    1. 判断reaction是不是PromiseReaction Record
    2. 声明promiseCapability为reaction.[[Capability]]
    3. 声明type为reaction.[[Type]]
    4. 声明handler为reaction.[[Handler]]
    5. 如果handler为空，则
      1. 如果type是Fulfill，则声明handlerResult为NormalCompletion(argument)(返回一个正常完成的记录)
      2. 其他情况，
        1. 判断type是不是Reject
        2. 声明handlerResult为ThrowCompletion(argument)(返回一个异常完成的记录)
    6. 其他情况，声明handlerResult为HostCallJobCallback(handler, undefined, <<argument>>)的执行结果（执行handler，并得到handler的执行结果）
    7. 如果promiseCapability为undefined，则
      1. 判断handlerResult是不是abrupt completion（判断handler执行过程是否有异常），如果不是
      2. 返回NormalCompletion(empty)
    8. 如果promiseCapability为Promise Capability Record，则
      1. 如果handlerResult是abrupt completion，则
        1. 声明status为Call(promiseCapability.[[Reject]], undefined, « handlerResult.[[Value]] »)的执行结果（把promise的状态设置为rejected，拒绝的原因是handlerResult的执行结果值）
      2. 其他情况，
        1. 声明status为Call(promiseCapability.[[Resolve]], undefined, « handlerResult.[[Value]] »)的执行结果（把promise的状态设置为rfulfilled，完成的结果是handlerResult的执行结果值）
    9. 返回Completion(status)
  2. 声明handlerRealm为null
  3. 如果reaction.[[Handler]]不为空，则
    1. 声明getHandlerRealmResult为GetFunctionRealm(reaction.[[Handler]].[[Callback]])的执行结果(获取handler的域)
    2. 如果getHandlerRealmResult的值是normal completion，设置handlerRealm为getHandlerRealmResult.[[Value]]的值
    3. 其他情况，则设置handlerRealm为the current Realm Record(将当前执行上下文的域)
    4. 备注：handlerRealm永远不为null，除非handler是undefined，当handler是废弃的Proxy并且没有ECMAScript代码在执行，handlerRealm用来创建错误对象
  4. 返回Record{ [[Job]]: job, [[Realm]]: handlerRealm }(创建好微任务，将包含微任务的Record返回)


## 27.2.3 The Promise Constructor

Promise构造器：

    - 是规范中语法中明确定义的对象Promise
    - 是global Object的属性Promise的初始值
    - 不能被当作函数使用，如果这样做了会抛出错误
    - 是可以用来继承的。在定义类时可以跟在extends后面使用。子类为了继承Promise的行为，必须在构造器内使用super方法去调用Promise的构造器，这样可以初始化子类实例为了支持Promise和Promise.prototype中的内建方法所必要的内部状态

### 27.2.3.1 Promise(executor)

当promise函数被调用，并且传入参数executor，以下步骤会执行：

    1. 如果NewTarget是undefined，抛出TypeError错误（未使用new的情况）
    2. 使用IsCallable判断executor是否可以执行，如果不能执行，抛出TypeError错误
    3. 声明promise，值是OrdinaryCreateFromConstructor(NewTarget, "%Promise.prototype%", « [[PromiseState]], [[PromiseResult]], [[PromiseFulfillReactions]], [[PromiseRejectReactions]], [[PromiseIsHandled]] »)的运行结果
    4. 设置promise.[[PromiseState]]为pending
    5. 设置promise.[[PromiseFulfillReactions]] 为空的List
    6. 设置promise.[[PromiseRejectReactions]] 为空的List
    7. 设置promise.[[PromiseHandleed]] 为false
    8. 声明resolvingFunctions，值是CreateResolvingFunctions(promise)的运行结果（结果是该promise的两个置值器）
    9. 声明completion，值是Call(executor, undefined, « resolvingFunctions.[[Resolve]], resolvingFunctions.[[Reject]] »)的执行结果（实际执行的是executor函数）
    10. 如果completion是abrupt completion（执行executor中出错），则
        1. 执行 Call(resolvingFunctions.[[Reject]], undefined, « completion.[[Value]] ») （实际执行的是promise的reject置值器，并将上一步的执行结果中的Value值作为rejct的reason）
    11. 返回promise

>executor必须是函数类型，调用它是为了代表新建的Promise去初始化和获取可能的延迟操作的完成。调用executor时会被传入resolve和reject两个参数。这些函数用于报告延迟操作是完成了或是失败了。executor返回了结果，并不意味着延迟操作完成了，只是意味请求执行延迟操作被接受了。
>
>传给executor的resolve函数接受一个参数。executor如果调用resolve函数，则表明它希望解决相关联的Promise。传给resolve的参数会被作为延迟操作的值，该参数既可以是实际完成值，也可以是另一个Promise对象状态为fulfilled时的值。
>
>传给executor的reject函数接受一个参数。executor如果调用reject函数，则表明它希望拒绝相关联的Promise，并且永远不会再变为fulfilled状态。传给reject的参数会被作为这个promise的拒绝值，一般是一个Error对象。
>
>传给executor的resolve和reject函数都有能力去真正的解决和拒绝相关联的promise。子类如果传入自定义的resolve和reject函数，构造器可能会有不同的表现。

## 27.2.4 Properties of the Promise Constructor

Promise构造器是：
  - 是规范中明确定义的对象Function.prototype，拥有[[Prototype]]内部插槽
  - 有以下的属性

### 27.2.4.6 Promise.reject(r)

reject方法返回一个用传入的参数拒绝的promise

  1. C为this的值
  2. promiseCapability为使用NewPromiseCapality(C)的结果
  3. 执行Call(promiseCapabilit.[[Reject]], undefined, <<r>>)(设置新创建的promise状态为rejected)
  4. 返回promiseCapability.[[Promise]](返回新创建的promise)

>为满足Promise构造器的参数要求，reject方法期望它的this值是一个构造函数

### 27.2.4.7

resolve方法返回一个用传入的参数解决的promise，如果传入的参数是一个promise，则会返回该参数

  1. C为this的值
  2. 使用Type(C)判断C是否是Object类型，如果不是，则抛出TypeError错误
  3. 返回执行PromiseResolve(C, x)的结果（返回一个解决的promise）

>为满足Promise构造器的参数要求，reject方法期望它的this值是一个构造函数

## 27.2.5 Properties of the Promise Prototype Object

Pomise原型对象：

 - 是规范中明确定义的对象Promise.prototype
 - 有[[Prototype]]内部插槽，值是Object.prototype
 - 是一个普通对象
 - 任何Promise实例中该对象都不含有[[PromiseState]]及其他内部插槽

### 27.2.5.4 Promise.prototype.then(onFulfilled, onRejected)

传入参数onFulfilled和onRejected调用then方法时，会执行以下步骤：

  1. promise是this的值
  2. 使用IsPromise(promise)，判断promise是不是Promise的实例，如果不是，则抛出TypeError错误
  3. C是SpeciesConstructor(promise, %Promise%)的执行结果（C是promise构造器）
  4. resultCapability是NewPromiseCapability(C)的执行结果（创建了新的Promise Record，也就是新的promise）
  5. 执行 PerformPromiseThen(promise, onFulfilled, onRejected, resultCapability)，并将执行结果返回

#### 27.2.5.4.1  PerformPromiseThen ( promise, onFulfilled, onRejected [ , resultCapability ] )

PerformPromiseThen抽象操作需要promise，onFulfilled，onRejected等参数，以及一个可选参数resultCapability（一个PromiseCapability Record）。在promise上执行then操作是将onFulfilled和onRejected作为它在敲定后的执行动作。如果传入了resultCapability，执行结果会保存在它的promise上。如果没有传，则会使用特定的内部操作去执行该操作，并且不关心执行的结果。当被调用时，会执行下方步骤：

  1. 使用Ispromise(promise)判断promise是否为true
  2. 如果resultCapability不存在，则
    1. 设置resultCapability为undefined
  3. 使用IsCallable(onFulfilled)是否可被调用，如果是false，则
    1. 声明onFulfilledJobCallback是空
  4. 反之，
    1. 声明onFulfilledJobCallback，值是HostMakeJobCallback(onfulfilled)的执行结果
  5. 使用IsCallable(onRejected是否可被调用，如果是false，则
    1. 声明onRejectedJobCallback是空
  6. 反之，
    1. 声明onRejectedJobCallback，值是HostMakeJobCallback(onRejected)的执行结果
  7. 声明fulfillReaction是PromiseReaction对象{[[Capability]]: resultCapability, [[Type]]: Fulfill, [[Handler]]: onFulfilledJobCallback}
  8. 声明rejectReaction是PromiseReaction对象{[[Capability]]: resultCapability, [[Type]]: Reject, [[Handler]]: onRejectedJobCallback}
  9. 如果promise.[[PromiseState]] 是pending，则
    1. 将fulfillReaction添加到promise.[[PromiseFulfillReactions]]这个列表中的最后
    2. 将rejectReaction添加到promise.[[PromiseRejectReactions]]这个列表中的最后
  10. 如果promise.[[PromiseState]] 是fulfilled，则
    1. 声明value,值是promise.[[PromiseResult]]
    2. 声明fulfillJob，值是NewPromiseReactionJob(fulfillReaction, value)执行结果(创建微任务)
    3. 执行HostEnqueuePromiseJob(fulfillJob.[[Job]], fulfillJob.[[Realm]])（将任务添加到微任务队列，Realm是域的意思）
  11. 其他情况，
    1. 判断promise.[[PromiseState]]是不是rejected
    2. 声明reason，值是pomise.[[PromiseResult]]
    3. 如果promise.[[PromiseIsHandled]] 是false，则执行HostPromiseRejectionTracker(promise, "handle")（告诉宿主环境，有未处理的rejected promise）
    4. 声明rejectJob，值是NewPromiseReactionJob(rejectReaction, reason)(创建微任务)
    5. 执行HostEnqueuePromiseJob(rejectJob.[[Job]], rejectJob.[[Realm]])（将任务添加到微任务队列，Realm是域的意思）
  12. 设置promise.[[PromiseIsHandled]]为true
  13. 如果resultCapability为undefined，则
    1. 返回undefined
  14. 其他情况
    1. 返回resultCapability.[[Promise]]