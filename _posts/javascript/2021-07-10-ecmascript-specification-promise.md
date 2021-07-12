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