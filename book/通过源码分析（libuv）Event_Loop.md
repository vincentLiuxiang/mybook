# 通过源码分析（libuv）Event Loop



写node.js有一段时间了，一直在理解event loop这个概念，国内外的文章翻阅了也不少，但是对event loop能讲解清楚的还是不多。


写这篇文章的目的是将自己对`eventloop`的理解以及`libuv`源码分析做一个记录，官方文档链接[The Node.js Event Loop, Timers, and process.nextTick()](https://github.com/nodejs/node/blob/master/doc/topics/the-event-loop-timers-and-nexttick.md#the-nodejs-event-loop-timers-and-processnexttick)

* 术语约束:
* 文中的timer特指，setTimeout、setInterval;
* setImmediate不是timer;


### Event Loop的解释
英文原文：
*When Node.js starts, it initializes the event loop, processes the provided input script (or drops into the REPL, which is not covered in this document) which may make async API calls, schedule timers, or call process.nextTick(), then begins processing the event loop.*

> 当Node.js启动时会初始化event loop, 每一个event loop都会包含按如下顺序六个循环阶段，

```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```
* **timers 阶段:** 这个阶段执行setTimeout(callback) and setInterval(callback)预定的callback;
* **I/O callbacks 阶段:** 一般是一些系统调用，比如tcp连接失败error的，callback;
* **idle, prepare 阶段:** 仅node内部使用;
* **poll 阶段:** 获取新的I/O事件, 适当的条件下node将阻塞在这里;
* **check 阶段:** 执行setImmediate() 设定的callbacks;
* **close callbacks 阶段:** 比如socket.on('close', callback)的callback会在这个阶段执行.


每一个阶段都有一个装有callbacks的fifo queue(队列)，当event loop运行到一个指定阶段时，
node将执行该阶段的fifo queue(队列)，当队列callback执行完或者执行callbacks数量超过该阶段的上限时，
event loop会转入下一下阶段.

##### `注意上面六个阶段都不包括 process.nextTick()，稍后做解释；`



### libuv 源码简单分析

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    // 更新当前event-loop事件戳
    // loop->time
    uv__update_time(loop);
    // timer阶段，执行timer heap,
    // timer heap, 你可以认为是根据定时器执行是时间排序的降序序链表
    // uv__run_timers 循环遍历timer_heap, 假设每一个待处理的timer事件对象为handle 
    // 1. 如果当前 handle->time >= loop->time, 执行handle->cb()
    // 2. 如果当前 handle->time < loop->time, 循环break, 
    // 为什么break? 是因为timer_heap是有序的
    uv__run_timers(loop);
    // 执行i/o callback;
    ran_pending = uv__run_pending(loop);
    // idle阶段
    uv__run_idle(loop);
   	// prepare阶段
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);
	
    // poll阶段, 里面其实就是epoll模型的核心，下次再写一篇epoll的文档，具体展开来讲
    uv__io_poll(loop, timeout);
    // check阶段
    uv__run_check(loop);
    // close阶段
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      // 如果代码只是执行一次，进程就退出，进程推出前再执行一次，uv__run_timers
      // 很好理解，因为上面第一次uv__run_timers，可能timer事件还未ready
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```



### poll阶段——uv__io_poll

poll阶段是衔接整个event loop各个阶段比较重要的阶段，为了便于后续例子的理解，本文先讲这个阶段；

在node.js里，任何异步方法（除timer,close,setImmediate之外）完成时，都会将其callback加到poll queue里,并立即执行。

**poll** 阶段有两个主要的功能：

1. 处理poll队列（poll quenue）的事件(callback);
2. 执行timers的callback,当到达timers指定的时间时;

如果event loop进入了 poll阶段，且代码未设定timer，将会发生下面情况：

* 如果poll queue不为空，event loop将同步的执行queue里的callback,直至queue为空，或执行的callback到达系统上限;
* 如果poll queue为空，将会发生下面情况：
* 如果代码已经被setImmediate()设定了callback, event loop将结束poll阶段进入check阶段，并执行check阶段的queue (check阶段的queue是 setImmediate设定的)
* 如果代码没有设定setImmediate(callback)，event loop将阻塞在该阶段等待callbacks加入poll queue;

如果event loop进入了 poll阶段，且代码设定了timer：

* 如果poll queue进入空状态时（即poll 阶段为空闲状态），event loop将检查timers,如果有1个或多个timers时间时间已经到达，event loop将按循环顺序进入 timers 阶段，并执行timer queue.


以上便是整个event loop时间循环的各个阶段运行机制，有了这层理解，我们来看几个例子

* 注意，例子中的时间是不同机器，同一机器不同执行时间都会有差异的。

### example 1
```js
var fs = require('fs');

function someAsyncOperation (callback) {
  // 花费2毫秒
  fs.readFile(__dirname + '/' + __filename, callback);
}

var timeoutScheduled = Date.now();
var fileReadTime = 0;

setTimeout(function () {
  var delay = Date.now() - timeoutScheduled;
  console.log('setTimeout: ' + (delay) + "ms have passed since I was scheduled");
  console.log('fileReaderTime',fileReadtime - timeoutScheduled);
}, 10);

someAsyncOperation(function () {
  fileReadtime = Date.now();
  while(Date.now() - fileReadtime < 20) {

  }
});
```
结果: 先执行someAsyncOperation的callback,再执行setTimeout callback

```shell
-> node eventloop.js
setTimeout: 22ms have passed since I was scheduled
fileReaderTime 2
```
解释：
当时程序启动时，event loop初始化：

1. timer阶段（无callback到达，setTimeout需要10毫秒）
2. i/o callback阶段，无异步i/o完成
3. 忽略
4. poll阶段，阻塞在这里，当运行2ms时，fs.readFile完成，将其callback加入 poll队列，并执行callback，
   其中callback要消耗20毫秒,等callback之行完，poll处于空闲状态，由于之前设定了timer，因此检查timers,发现timer设定时间是20ms，当前时间运行超过了该值，因此，立即循环回到timer阶段执行其callback,因此，虽然setTimeout的20毫秒，但实际是22毫秒后执行。

### example 2
```
var fs = require('fs');

function someAsyncOperation (callback) {
  var time = Date.now();
  // 花费9毫秒
  fs.readFile('/path/to/xxxx.pdf', callback);
}

var timeoutScheduled = Date.now();
var fileReadTime = 0;
var delay = 0;

setTimeout(function () {
  delay = Date.now() - timeoutScheduled;
}, 5);

someAsyncOperation(function () {
  fileReadtime = Date.now();
  while(Date.now() - fileReadtime < 20) {

  }
  console.log('setTimeout: ' + (delay) + "ms have passed since I was scheduled");
  console.log('fileReaderTime',fileReadtime - timeoutScheduled);
});
```
结果：setTimeout callback先执行，someAsyncOperation callback后执行

```js
-> node eventloop.js
setTimeout: 7ms have passed since I was scheduled
fileReaderTime 9
```
解释：
当时程序启动时，event loop初始化：

1. timer阶段（无callback到达，setTimeout需要10毫秒）
2. i/o callback阶段，无异步i/o完成
3. 忽略
4. poll阶段，阻塞在这里，当运行5ms时，poll依然空闲，但已设定timer,且时间已到达，因此，event loop需要循环到timer阶段,执行setTimeout callback,由于从poll --> timer中间要经历check,close阶段,这些阶段也会消耗一定时间，因此执行setTimeout callback实际是7毫秒 然后又回到poll阶段等待异步i/o完成，在9毫秒时fs.readFile完成，其callback加入poll queue并执行。


### setTimeout 和 setImmediate
二者非常相似，但是二者区别取决于他们什么时候被调用.

* setImmediate 设计在poll阶段完成时执行，即check阶段；
* setTimeout 设计在poll阶段为空闲时，且设定时间到达后执行；但其在timer阶段执行

其二者的调用顺序取决于当前event loop的上下文，如果他们在异步i／o callback之外调用，其执行先后顺序是不确定的

```js
setTimeout(function timeout () {
  console.log('timeout');
}, 0);

setImmediate(function immediate () {
  console.log('immediate');
});
```

```
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```
解释：

二者执行的先后顺序是不一定的，由系统执行状态决定。具体原因，我简单引入一下libuv的c语言代码：

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  
  ...

  while (r != 0 && loop->stop_flag == 0) {
    // 更新当前event-loop事件戳
    // loop->time
    uv__update_time(loop);
    // timer阶段，执行timer heap,
    // timer heap, 你可以认为是根据定时器执行是时间排序的降序序链表
    // 这里涉及一个问题:
    // 1. eventloop各个阶段的queue, 是在运行uv_run前注册(添加)进去的。当前在eventloop执行过程中，也可以往各个阶段的queue里添加cb.
    // 2. timer_heap也是在uv_run之前就创建了
    // 3. 比如：
       		/*
       			setTimeout(function timeout () {
                  console.log('timeout');
                },0);
       		*/	
    // 4. 在3的例子中，在调用setTimeout时其实是往timer_heap注册一个handle结构体，handle里包含cb, time等信息， 针对例子3，handle->time = system_current_time + 1;
    // 5. 当系统运行到uv__update_time(loop)时, 因为cpu时间片分配的不确定性与绝对时间一定性，会造成
    // 调用uv__update_time(loop)造成 loop->time 大于／等于／小于 handle-> time 均有可能
    uv__run_timers(loop);
	
    ...
```



但当二者在异步i/o callback内部调用时，总是先执行setImmediate，再执行setTimeout

```
var fs = require('fs')

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout')
  }, 0)
  setImmediate(() => {
    console.log('immediate')
  })
})
```

```
$ node timeout_vs_immediate.js
immediate
timeout
```
理解了event loop的各阶段顺序这个例子很好理解：
因为fs.readFile callback执行完后，程序设定了timer 和 setImmediate，因此poll阶段不会被阻塞进而进入check阶段先执行setImmediate，后进入timer阶段执行setTimeout

### process.nextTick()
千呼万唤始出来，终于到了讲process.nextTick()的时候，来来来喝口水休息休息。

注意：本文讲解的process.nextTick是基于v0.10及以上版本

**process.nextTick()不在event loop的任何阶段执行，而是在各个阶段切换的中间执行**,即从一个阶段切换到下个阶段前执行。

```
var fs = require('fs');

fs.readFile(__dirname, () => {
  setTimeout(() => {
    console.log('setTimeout');
  }, 0);
  setImmediate(() => {
    console.log('setImmediate');
  });
  process.nextTick(()=>{
    console.log('nextTick');
  })
});
```

```
-> node eventloop.js
nextTick
setImmediate
setTimeout
```
从poll ---> check阶段，先执行process.nextTick，然后进入check,setImmediate，最后进入timer执行setTimeout

process.nextTick()是node早期版本无setImmediate时的产物，node作者推荐我们尽量使用setImmediate。

### example process.nextTick()
当递归调用process.nextTick时，即使fs.readFile完成，其callback无机会执行

```
var fs = require('fs');
var starttime = Date.now();
var endtime = 0;

fs.readFile(__filename, () => {
  endtime = Date.now();
  console.log('finish reading time:',endtime - starttime);
});

var index = 0;

function nextTick () {
  if (index > 1000) return;
  index++;
  console.log('nextTick');
  process.nextTick(nextTick);
}

nextTick();
```

```
-> node eventloop.js
nextTick
nextTick
...
nextTick
nextTick
finish reading time: 246
```

### example setImmediate
将process.nextTick替换成setImmediate后，由于setImmediate只在check阶段执行，那么所有的callback都有机会执行。

```
var fs = require('fs');

fs.readFile(__filename, () => {
  console.log('finish reading');
});

var index = 0;

function Immediate () {
  if (index > 100) return;
  index++;
  console.log('setImmediate');
  setImmediate(Immediate);
}

Immediate()
```


```
-> node eventloop.js
setImmediate
setImmediate
setImmediate
setImmediate
finish reading time: 19
...
setImmediate
setImmediate
```



### 文档最后再提一下 node EventEmitter

node events是同步的！

```js
var events = require('events');
var emitter = new events.EventEmitter();

emitter.on('someEvent', function(arg1, arg2) {
	console.log('listener');
});

emitter.emit('someEvent');
```



events注册的事件, 其实是存在了一个event_queue里，伪代码分析如下：

```js
// 以下是伪代码
e.on('test', cb);
// 等价于 eventqueue['test'].push(cb)
e.emit('test');
// 等价于 eventqueue['test'].map(cb => cb())，这是伪代码
```