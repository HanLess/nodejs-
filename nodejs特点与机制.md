对比 java，nodejs比较显著的特点有三点：

（1）单进程，单线程

（2）异步，非阻塞IO

（3）事件驱动


### 单进程，单线程

java有多线程机制，各种sevlet容器（如tomcat）在处理请求的时候会维护一个线程池来应对高并发，即一个http请求对应一条线程，
而维护一条线程的成本很高，所以并发高了就要堆机器

nodejs 没有多线程的机制，因为是V8引擎驱动，V8是用在客户端的，不能开多线程。所以nodejs程序是单进程（子进程属于补丁设计），单线程的。

注：这里的单线程指的是 javascript 代码的执行只在一个线程里（主线程），但涉及到IO操作还是会另开线程的

node没有多线程，该怎么处理高并发？

这里使用了异步的实现思想：使用任务队列管理并发请求，执行栈会依次处理任务队列中的任务

优点：因为不用维护多线程，会节省很多资源，所以在同样的机器下，node可以处理更多请求

缺点：

（1）使用任务队列也需要资源，且如果并发过高，单线程无法及时处理，也会造成等待时间过长（高并发处理乏力）

（2）所有任务都在一个线程中执行，一旦某个节点有问题，整个程序都会崩溃（稳定性）

解决办法：Nginx代理做负载均衡，使用集群管理（pm2）

### 异步，非阻塞IO

其实java使用多线程也可以实现异步IO，且node实现异步IO也是另开了线程，可以理解为nodejs的语法糖

当nodejs主线程需要执行IO操作时，会把IO操作分配给IO线程执行，主线程则继续执行下面的逻辑，这里以 http 请求距离：

```
场景：客户端进行http请求，nodejs处理数据，查数据库，返回数据给客户端

流程：

接收到http请求后，nodejs主线程开始工作，计算数据，执行业务逻辑，执行到查库的代码后，另开线程执行查表动作，主线程继续执行直到执行栈清空

这是如果查表动作还没有结束，会继续处理下一个http请求（http请求全在任务队列中），逻辑同上

这时候如果上一个http请求的查表逻辑执行结束，就会执行回调，把查到的数据返回给上一个请求的客户端，然后继续执行主线程逻辑
```

### 事件驱动

主要类：EventEmitter 

nodejs中大部分类（实现异步操作的）都继承了 EventEmitter 类，可以通过 on，emit来监听，触发自定义事件

如果要实现异步操作，就必须通过这个类来实现对异步操作的监听，当异步操作完成后，通过emit来触发回调函数的执行

### nodejs中的event loop

参考文章：https://juejin.im/post/5b5dcb3e5188257bcb59083f

nodejs中的事件循环与普通js中的处理大致相同：执行同步代码 -> 异步事件推入事件队列

在清空执行事件队列时，仍然遵循先清空微事件，再执行一个宏事件

nodejs中的任务队列种类：

```
宏观事件：
Timers Queue - 计时器队列
I/O Queue - 输入输出队列
Check Queue - 检查队列
Close Queue - guangbi 队列

微观事件
MircoTask Queue  - 浏览器和nodejs共有的微任务（micro task）
NextTick Queue  - node的 process.nextTick
```

宏观队列也有执行顺序（从上到下循环）：

```
┌───────────────────────────────────┐
┌─>│timers(计时器)执行                  │
│  |setTimeout以及setInterval的回调     │
│  └──────────┬────────────────────────┘
┌──────────————┴────────────┐
│  │     I/O callbacks     │
│   处理网络，流，TCP的错误  │  
│  │      callback      │
│  └──────────┬────────────┘
┌──────────┴────────────┐
│  │     idle, prepare     │
│  │    node内部使用        │
│  └──────────┬────────────┘          
 ┌──────────┴────────────┐      │   incoming:   │
│  │poll（轮询）            │<─────┤  connections, │
│  │ 执行poll中的i/o队列检查 │      │data, etc.    │
│  │定时器是否到时          │      └───────────────┘
│  └──────────┬────────────┘              
│  ┌──────────┴────────────┐      
│  │        check          │
│  │  存放setImmediate回调  │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   │ 关闭的回调例如         │
   │ socket.on('close')    │
   └───────────────────────┘
```

示例图片：<img src="https://github.com/HanLess/nodejs-analysis/blob/master/nodejs-event-loop.png" />

示例代码：

```
// 清空TimerQueue
setTimeout(...）  
// 清空该进程中的微任务
// then1位置的Promise先进入任务队列
Promise.resolve().then(()=>{ 
   console.log('then1'); // then1
});
Promise.resolve().then(()=>{
    console.log('then2'); // then2
})
// 接着进入IO队列
fs.readFile(...)
// 优先清空IO队列的NextTick Queue
process.nextTick(function(){
   console.log('nextTick') // nextTick
})
// 清空micro queue
setImmediate(()=>{
   console.log('setImmediate')//setImmediate
});


/*
执行顺序：
setTimeout
Promise.resolve().then
fs.readFile

输出：
then1
then2
nextTick
setImmediate
*/
```


