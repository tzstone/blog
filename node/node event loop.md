# node event loop

虽然 JavaScript 是单线程的, 但是通过尽可能将操作转移(offloading)到系统内核, 事件循环(event loop)允许 Node.js 执行非阻塞 I/O 操作.

由于大多数现代内核都是多线程的，因此它们可以处理在后台执行的多个操作。当其中一个操作完成时，内核会通知 Node.js，以便可以将相应的回调添加到轮询队列中以最终执行。

注: node 源码的版本为 10.9.0

**event loop 顺序**:

![](https://github.com/tzstone/MarkdownPhotos/blob/master/node-event-loop.jpeg)

每个方框对应 event loop 的一个阶段.
每个阶段有一个 FIFO 队列, 用来存放待执行的 callback. 当 event loop 进入某个给定的阶段时, 会执行该阶段的一些特定操作, 然后执行该阶段队列里的 callback, 直到队列执行完或者执行的 callback 数量达到最大值, 此时 event loop 将移到下一阶段, 以此类推.

- `timers`: 这个阶段执行`setTimeout()`和`setInterval()`设定的 callback. 从技术上讲，`poll`阶段控制何时执行定时器。
- `pending callbacks`: 执行某些系统操作的设定的 callback(如 TCP 错误)
- `idle, prepare`: 仅在 node 内部使用.
- `poll`: 获取新的 I/O 事件; 执行与 I/O 相关的设定的 callback; 适当条件下 node 将堵塞在这里.
- `check`: 执行`setImmediate()`设定的 callback
- `close callbacks`: 一些 close callback 将在这里执行, 如果`socket.on('close', callback)`

event 是由 `uv_run` 驱动的, 并且是在 `UV_RUN_ONCE` 模式下执行的. `uv_run` 有另外两种模式 `UV_RUN_DEFAULT` 和 `UV_RUN_NOWAIT`.
由源码可知, 在进入 poll 阶段前会计算 timeout 并将 timeout 传入 `uv__io_poll`. [timeout 的计算规则](http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop)如下:

- 如果 loop 运行在 `UV_RUN_NOWAIT` 模式下, timeout 为 0.
- 如果 loop 将要被停止 (如调用了 uv_stop()), timeout 为 0.
- 如果没有活动的 handles 或者 requests, timeout 为 0.
- 如果有活动的 idle handles, timeout 为 0.
- 如果有任何等待被 close 的 callbacks(`close callbacks`阶段的回调), timeout 为 0.
- 如果上述条件都不匹配, timeout 会取最近的一个定时器的剩余超时时间, 如果没有活动的定时器, timeout 则被设置为无限大.

当 timeout 不为 0 时, event loop 就会堵塞在 poll 阶段等待 callback 的到来. 这样 poll 在等待时, 如果没有任何 I/O 事件触发, 也会由 timeout 超时跳出等待.

另外, 在 UV_RUN_ONCE 模式下, 每次循环结束前(`close callbacks`执行结束后), 会调用`uv__run_timers(loop);`执行对 timer 的超时判断, 所以实际上在一次 event loop 中, timer 有两个可能执行的地方: 最开始的`timer`阶段及`close callbacks`阶段之后--这导致了有时候`setImmediate()`和`setTimeout()`的执行顺序是不确定的.

```js
// uv_run源码注解
// 来自 https://cnodejs.org/topic/57d68794cb6f605d360105bf 中 @hyj1991 的评论
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  //uv__loop_alive返回的是event loop中是否还有待处理的handle或者request
	//以及closing_handles是否为NULL,如果均没有,则返回0
  r = uv__loop_alive(loop);
  //更新当前event loop的时间戳,单位是ms
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    //(以mac为例)使用Linux下的高精度Timer hrtime更新loop->time,即event loop的时间戳
    uv__update_time(loop);
    //执行判断当前loop->time下有无到期的Timer,显然在同一个loop里面timer拥有最高的优先级
    uv__run_timers(loop);
    //判断当前的pending_queue是否有事件待处理,并且一次将&loop->pending_queue中的uv__io_t对应的cb全部拿出来执行
    ran_pending = uv__run_pending(loop);
    //实现在loop-watcher.c文件中,一次将&loop->idle_handles中的idle_cd全部执行完毕(如果存在的话)
    uv__run_idle(loop);
    //实现在loop-watcher.c文件中,一次将&loop->prepare_handles中的prepare_cb全部执行完毕(如果存在的话)
    uv__run_prepare(loop);

    timeout = 0;
    // 计算轮询超时时间
    //如果是UV_RUN_ONCE的模式,并且pending_queue队列为空,或者采用UV_RUN_DEFAULT(在一个loop中处理所有事件),则将timeout参数置为
    //最近的一个定时器的超时时间,防止在uv_io_poll中阻塞住无法进入超时的timer中
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    //进入I/O处理的函数(重点分析的部分),此处挂载timeout是为了防止在uv_io_poll中陷入阻塞无法执行timers;并且对于mode为
    //UV_RUN_NOWAIT类型的uv_run执行,timeout为0可以保证其立即跳出uv__io_poll,达到了非阻塞调用的效果
    uv__io_poll(loop, timeout);
    //实现在loop-watcher.c文件中,一次将&loop->check_handles中的check_cb全部执行完毕(如果存在的话)
    uv__run_check(loop);
    //执行结束时的资源释放,loop->closing_handles指针指向NULL
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
      //如果是UV_RUN_ONCE模式,继续更新当前event loop的时间戳
      uv__update_time(loop);
      //执行timers,判断是否有已经到期的timer
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    //在UV_RUN_ONCE和UV_RUN_NOWAIT模式中,跳出当前的循环
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  //标记当前的stop_flag为0,表示当前的loop执行完毕
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```

## `poll`阶段

有两个重要的功能:

- 计算它应该堵塞和轮询 I/O 的时间
- 处理轮询队列(poll queue)中的事件(callback)

### 当 event loop 进入 poll 阶段且代码未设定 timer 时

- poll 队列非空: event loop 将遍历执行(同步) poll 队列中的 callback, 直到遍历完成或者达到系统相关的限制.
- poll 队列为空:
  - 如果脚本已经被`setImmediate()`设定了 callback，则 event loop 将结束 poll 阶段并进入 check 阶段, 以执行这些 callback。
  - 如果脚本没有`setImmediate()`设定的 callback，则 event loop 将堵塞在该阶段, 等待 callback (如传入连接(incoming connections)，请求等)被添加到 poll queue，然后立即执行它们。

### 当 event loop 进入 poll 阶段且代码设定了 timer 时

一旦 poll 队列为空，event loop 将检查已达到时间阈值的计时器。如果一个或多个计时器准备就绪，event loop 将按循环顺序回绕到 `timers` 阶段(会经过`check`, `close callbacks`阶段)以执行那些计时器的 callback。

- 例子 1

```js
var fs = require('fs');

function someAsyncOperation(callback) {
  // 耗时2毫秒
  fs.readFile('./index.js', callback);
}

var timeoutScheduled = Date.now(); // 起始时间
var fileReadTime = 0;

setTimeout(function() {
  var delay = Date.now() - timeoutScheduled; // setTimeout延迟时间
  console.log('setTimeout: ' + delay + 'ms have passed since I was scheduled');
  console.log('fileReaderTime', fileReadtime - timeoutScheduled);
}, 10);

someAsyncOperation(function() {
  fileReadtime = Date.now(); // 读取文件结束时间
  while (Date.now() - fileReadtime < 20) {}
});

// result
setTimeout: 22ms have passed since I was scheduled
fileReaderTime 2
```

解释:

1.  程序启动, event loop 初始化, 执行脚本. 脚本设定了 timer(setTimeout), 开启文件异步读取, 然后开始执行 event loop
2.  `timer` 阶段, 没有就绪的 timer(setTimeout 需要 10ms 才就绪), 进入下一阶段
3.  `pending callbacks`阶段, 无系统操作的 callback, 进入下一阶段
4.  `idle, prepare`阶段, 略过, 进入下一阶段
5.  `poll`阶段, event loop 堵塞在该阶段, 等待 callback. 2ms 后 fs.readFile 完成, 将 callback 加入 poll queue 并立即执行, callback 耗时 20ms. callback 执行完, poll 处于空闲状态, 检查 timers. 由于 timer 在 10ms 的时候就已经就绪了, 因此 event loop 会回绕到`timer`阶段(经过`check`, `close callbacks`阶段)
6.  下一个 loop 的`timer`阶段, 执行 timer 的 callback, 此时已经过了 22ms(文件读取 2ms + 执行 callback20ms)

- 例子 2

```js
var fs = require('fs');

function someAsyncOperation(callback) {
  // 花费10毫秒
  fs.readFile('./Web性能权威指南.pdf', callback);
}

var timeoutScheduled = Date.now();
var fileReadTime = 0;
var delay = 0;

setTimeout(function() {
  delay = Date.now() - timeoutScheduled;
}, 5);

someAsyncOperation(function() {
  fileReadtime = Date.now();
  while (Date.now() - fileReadtime < 20) {}
  console.log('setTimeout: ' + delay + 'ms have passed since I was scheduled');
  console.log('fileReaderTime', fileReadtime - timeoutScheduled);
});

// result
setTimeout: 6ms have passed since I was scheduled
fileReaderTime 10
```

解释:

1.  程序启动, event loop 初始化, 执行脚本, 然后开始执行 event loop
2.  `timer` 阶段, 没有就绪的 timer(setTimeout 需要 5ms 才就绪), 进入下一阶段
3.  `pending callbacks`阶段, 无系统操作的 callback, 进入下一阶段
4.  `idle, prepare`阶段, 略过, 进入下一阶段
5.  `poll`阶段, event loop 堵塞在该阶段, 等待 callback. 5ms 时 timer 准备就绪, 此时 poll 仍然是空闲的, 因此 event loop 会回绕到`timer`阶段(经过`check`, `close callbacks`阶段)
6.  下一个 loop 的`timer`阶段, 执行 timer 的 callback, 此时已经过了 6ms(setTimeout 延时 5ms + `check`和`close callbacks`阶段的消耗), callback 执行完后, 经过`pending callbacks`阶段和`idle, prepare`阶段, 进入`poll`阶段
7.  下一个 loop 的`poll`阶段, event loop 堵塞在该阶段, 等待 callback. 10ms 时 fs.readFile 完成, 将 callback 加入 poll queue 并立即执行

## `setImmediate()` vs `setTimeout()`

- setImmediate: 一旦当前 poll 阶段完成便立即执行
- setTimeout: 经过最小时间阈值(ms)后运行

注: 在 node 中，setTimeout(cb, 0) === setTimeout(cb, 1)

定时器的执行顺序根据调用它们的上下文而有所不同.如果从主模块中调用两者，则时间将受到进程性能的限制（可能受到计算机上运行的其他应用程序的影响）。
例如，如果在非 I/O 周期内(比如主模块)运行脚本，则执行两个定时器的顺序是不确定的，因为它受进程性能的约束.

```js
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});

// 结果不确定
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

在 I/O 周期内运行, `setImmediate` 的回调总是比 `setTimeout` 先执行. 因为在 I/O 周期内设定了 `setImmediate` 和 `setTimeout` 的 callback 后, 当 poll queue 为空时, event loop 将按顺序分别进入 `check` 和 `timer` 阶段执行设定的 callback.

```js
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});

// setImmediate先执行
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

## `process.nextTick()`

从技术上讲, `process.nextTick()`不是 event loop 的一部分。相反，`nextTickQueue`将在执行完当前阶段的 queue，进入下一阶段前处理(即两个阶段的中间), 而不管 event loop 当前处于哪个阶段。这很容易造成问题--当递归调用`process.nextTick()`时, event loop 无法进入 poll 阶段, I/O 会被堵塞.

使用`node index.js`命令运行脚本时, 最终会进入 node 源码`./lib/internal/modules/cjs/loader.js`中的`Module.runMain`方法, 脚本 load 后会调用`process._tickCallback();`清空 nextTickQueue.

- 例子 1

```js
const fs = require("fs");

fs.readFile("./index.js", () => {
  setTimeout(() => {
    console.log("setTimeout");
  }, 0);
  setImmediate(() => {
    console.log("setImmediate1");
    process.nextTick(() => {
      console.log("nextTick3");
    });
  });
  setImmediate(() => {
    console.log("setImmediate2");
    process.nextTick(() => {
      console.log("nextTick4");
    });
  });
  process.nextTick(() => {
    console.log("nextTick1");
  });
  process.nextTick(() => {
    console.log("nextTick2");
  });
});

// result
nextTick1;
nextTick2;
setImmediate1;
setImmediate2;
nextTick3;
nextTick4;
setTimeout;
```

解释:

1.  event loop 堵塞在 poll 阶段, 收到 fs.readFile 的完成通知, 将 callback 加入 poll queue 并执行, 并在 callback 中设定了`setTimeout`, `setImmediate`, `process.nextTick()`的 callback.

2.  当 fs.readFile 的回调执行完, 即 poll 空闲时, 由于设定了`setImmediate`, event loop 会进入`check`阶段.

3.  进入`check`阶段之前, 会先执行`process.nextTick()`的 callback, `输出 nextTick1, nextTick2`.

4.  进入`check`阶段, 执行`setImmediate`的 callback, `输出 setImmediate1, setImmediate2`, 并在 callback 中设定了`process.nextTick()`的 callback.

5.  结束`check`阶段, 进入`close callback`阶段之前, 执行`process.nextTick()`的 callback, `输出 nextTick3, nextTick4`.

6.  最后进入`timer`阶段执行`setTimeout`的 callback, `输出setTimeout`

- 例子 2

递归调用`process.nextTick`, event loop 无法进入 poll 阶段, I/O 会被堵塞

```js
const fs = require("fs");
var startTime = Date.now();
var count = 0;

fs.readFile("./index.js", () => {
  var delay = Date.now() - startTime;
  console.log("fileread: " + delay + " ms");
});

var nextTick = function() {
  if (++count > 1000) return;
  console.log("nextTick");
  process.nextTick(nextTick);
};

nextTick();

// result
nextTick
nextTick
nextTick
...
nextTick
nextTick
fileread: 109 ms
```

- 例子 3

递归调用`setImmediate`(递归调用`setTimeout`也是类似结果), 不会堵塞 io. 推荐使用`setImmediate`而不是`process.nextTick`.

猜测`setImmediate`和`setTimeout`是异步把 callback 放到各自阶段的 queue 里, 而`process.nextTick`是同步把 callback 放到 queue 里, 所以`process.nextTick`会堵塞 i/o 而另外两个不会.

```js
const fs = require("fs");
var startTime = Date.now();
var count = 0;

fs.readFile("./index.js", () => {
  var delay = Date.now() - startTime;
  console.log("fileread: " + delay + " ms");
});

var immediate = function() {
  if (++count > 1000) return;
  console.log("setImmediate");
  setImmediate(immediate);
};

immediate();

// result
setImmediate
setImmediate
...
setImmediate
fileread: 24 ms
...
setImmediate
setImmediate
```

## 参考资料

- [event-loop-timers-and-nexttick](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

- [Node.js Event Loop 的理解 Timers，process.nextTick()](https://cnodejs.org/topic/57d68794cb6f605d360105bf)

- [libuv Design overview](http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop)

- [node 源码详解（二 ）—— 运行机制 、整体流程【markdown 版本】](https://cnodejs.org/topic/56e3be21f5d830306e2f0fd3)
