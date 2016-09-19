# node.js connect框架中间件遍历方式的一点优化想法

一直喜欢用node.js原生的http模块来写web服务，connect增加了middleware stack的概念。用户在使用时，可以根据需要来定制自己的中间件,
中间件的request,responese,都是http模块相关类的实例,connect并没有做任何封装。

所以，现在的项目基本上都是基于connect写中间件。


> non-error middleware,function有三个入参，分别是 req,res,next

```
app.use('/router',(req,res,next) => {
})
```

> error middleware, function 有四个入参，分别是 err,req,res,next

```
app.use((err,req,res,next) => {
})
```

### 举个例子---connect中间件的遍历方式
```
var app = require('connect')();
//假设第一个中间件出错了
app.use((req,res,next) => {
  next(new Error('something error'));
})

//...
//假设中间还有49个non-error 中间件

//最后挂载一个错误中间件，会捕获任何前面任何中间件发出的next(new Error(...))
app.use((err,req,res,next) => {
  res.end(err.message);
})
```

* 上面例子connect middleware stack数组长度为 51，即50个非错误中间件，1个错误中间件，当第一个中间件调用next(new Error('something error'))后，第51个中间件（错误中间件）会捕获该错误

### 那么，connect是如何捕获该错误的呢？
* 当在第一个中间件调用next(new Error('something error'))后，此时js 主线程会去stack取出第二个中间件，并判断req.url是否和router匹配，假如匹配，再判断该中间件事否为错误中间件.
* 由于2-50的中间件都是非错误中间件，connect这次http请求会占用js 主线程，循环50次直到第51个中间件是路由匹配的错误中间件，最后才进入该错误中间件内部处理；


**上面例子可以看出，当某个中间件调用next(new Error('something error'))后，js main线程会遍历当前中间件后面（stack数组编号偏后）的所有中间件，直到找到符合路由规则的错误中间件。**

我认为这样的实现方式是很低效的。

# 优化方案
* 在connect 内部除了stack之外，增加errware数组，专门用于存放错误中间件;
* 调用app.use(middleware)时，就区分该中间件是错误中间件还是非错误中间件，非错误中间件push到stack,错误中间件push到errware;

```
 -  this.stack.push({ route: path, handle: handle });
 +  if (handle.length === 4) {
 +    this.errware.push({ route: path, handle: handle, index: this.stack.length });
 +  } else {
 +    this.stack.push({ route: path, handle: handle, index: this.errware.length });
 +  }
```
* push到stack数组的每个对象里都增加一个index字段，用于存放当前errware的长度，即该非错误中间件调用next(error)时，可直接索引到其后面的错误中间件，而忽略其前面注册的错误中间件；
* push到errware数组的每个对象也都增加一个index字段，用于存放当前stack的长度，即该错误中间件假如调用了next(),可以直接索引到其后面非错误中间件，而忽略其前面注册的非错误中间件；

```
 -    var layer = stack[index++];
 +    var layer;
 +    if (err) {
 +      layer = errware[errIndex++];
 +      if (layer) index = layer.index;
 +    } else {
 +      layer = stack[index++];
 +      if (layer) errIndex = layer.index;
 +    }
```
### [代码diff](https://github.com/senchalabs/connect/pull/1085/files)

### 还是上面的例子
```
var app = require('connect')();
//假设第一个中间件出错了
app.use((req,res,next) => {
  next(new Error('something error'));
})

//...
//假设中间有50个non-error 中间件

//最后挂载一个错误中间件，会捕获任何前面任何中间件发出的next(new Error(...))
app.use((err,req,res,next) => {
  res.end(err.message);
})
```

经过优化后，在第一个非错误中间件调用next(error)后，会跳过中间所有非错误中间件直接访问错误中间件，不需要js 主线程去循环遍历50次。

## benchMark

> 为了防止finalhandler 模块抱错，我的测试环境里吧这段代码屏蔽了

```
// all done
if (!layer) {
  // defer(done, err);  //commented in my test
  return;
}
```

### 性能测试脚本 1 

```
var Benchmark = require('benchmark');
var appNew = require('../../../../github/connect')();
var appOld = require('connect')();
var http = require('http');

var suite = new Benchmark.Suite;

function test (app,pre,errpre,next,errnext) {
  for (var i = 0; i < pre; i++) {
    app.use((req,res,next) => {
      next();
    });
  }

  for (var i = 0; i < errpre; i++) {
    app.use((err,req,res,next) => {
      // res.end(err.message);
    })
  }

  for (var i = 0; i < next; i++) {
    app.use((req,res,next) => {
      next(new Error('error'));
    });
  }

  for (var i = 0; i < errnext; i++) {
    app.use((err,req,res,next) => {

    })
  }

  return app;
}

/**
 * a bad benchmark case for appNew
 * if app does not have error-middleware,
 * and it will has no Performance optimization
 */
// appNew = test(appNew,50);
// appOld = test(appOld,50);

/**
 * a beneficial benchmark case for appNew
 * if app has error-middleware,
 * the Performance optimization rely on the number of error-middleware
 * and it's position when invoke app.use
 */
appNew = test(appNew,0,0,50,1);
appOld = test(appOld,0,0,50,1);

var req = {url:'/'};
var res = {};


console.log('start testing ...');

suite
.add('connectOld', function() {
  appOld(req,res);
})
.add('connectNew', function() {
  appNew(req,res)
})
.on('cycle', function(event) {
  console.log(String(event.target));
})
.on('complete', function() {
  console.log('Fastest is ' + this.filter('fastest').map('name'));
})
.run({ 'async': true });
```

### result
> 无错误中间件

> appNew = test(appNew,50);
 
> appOld = test(appOld,50);

> 性能差不多.

```
-> node benchmark.js
start testing ...
connectOld x 48,994 ops/sec ±4.75% (64 runs sampled)
connectNew x 51,117 ops/sec ±1.67% (81 runs sampled)
Fastest is connectNew,connectOld
```
>有错误中间件

>appNew = test(appNew,0,0,50,1);

>appOld = test(appOld,0,0,50,1);

> 优化后明显有提升

```
-> node benchmark.js
start testing ...
connectOld x 35,875 ops/sec ±5.24% (82 runs sampled)
connectNew x 145,842 ops/sec ±3.25% (79 runs sampled)
Fastest is connectNew
```


### 性能测试脚本 2

```
var appNew = require('../../../../github/connect')();
var appOld = require('connect')();
var express = require('express')();
var http = require('http');

appNew = test(appNew,0,0,50,1);
appOld = test(appOld,0,0,50,1);
express = test(express,0,0,50,1);

http.createServer((req,res) => {
  //appNew(req,res);
  //appOld(req,res);
  express(req,res);
}).listen(3002)
```
### result

* express

```
http.createServer((req,res) => {
  express(req,res);
}).listen(3002)
```

```
-> wrk -t 8 -c 100 -d 30 http://127.0.0.1:3002
Running 30s test @ http://127.0.0.1:3002
  8 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    14.77ms    2.53ms  71.22ms   83.84%
    Req/Sec   817.15    107.04     1.29k    76.00%
  195447 requests in 30.06s, 23.67MB read
Requests/sec:   6502.34
Transfer/sec:    806.44KB
```

* connect

```
http.createServer((req,res) => {
  appOld(req,res);
}).listen(3002)
```

```
-> wrk -t 8 -c 100 -d 30 http://127.0.0.1:3002
Running 30s test @ http://127.0.0.1:3002
  8 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    11.95ms    5.08ms  94.55ms   91.29%
    Req/Sec     1.04k   246.71     1.82k    74.08%
  247739 requests in 30.09s, 24.57MB read
Requests/sec:   8232.69
Transfer/sec:    836.13KB
```

* connect优化后

```
http.createServer((req,res) => {
  appNew(req,res);
}).listen(3002)
```

```
-> wrk -t 8 -c 100 -d 30 http://127.0.0.1:3002
Running 30s test @ http://127.0.0.1:3002
  8 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     8.53ms    2.16ms  66.62ms   93.58%
    Req/Sec     1.42k   176.98     2.41k    84.33%
  340279 requests in 30.06s, 33.75MB read
Requests/sec:  11319.54
Transfer/sec:      1.12MB
```







