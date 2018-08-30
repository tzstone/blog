# node event loop

虽然 JavaScript 是单线程的, 但是通过尽可能将操作转移(offloading)到系统内核, 事件循环(event loop)允许 Node.js 执行非阻塞 I/O 操作.

由于大多数现代内核都是多线程的，因此它们可以处理在后台执行的多个操作。当其中一个操作完成时，内核会通知 Node.js，以便可以将相应的回调添加到轮询队列中以最终执行。

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

## `setImmediate()` vs `setTimeout()`

- setImmediate: 一旦当前 poll 阶段完成便立即执行
- setTimeout: 经过最小时间阈值(ms)后运行

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

event loop 堵塞在 poll 阶段, 收到 fs.readFile 的完成通知, 将 callback 加入 poll queue 并执行, 并在 callback 中设定了`setTimeout`, `setImmediate`, `process.nextTick()`的 callback.

当 fs.readFile 的回调执行完, 即 poll 空闲时, 由于设定了`setImmediate`, event loop 会进入`check`阶段.

进入`check`阶段之前, 会先执行`process.nextTick()`的 callback, `输出 nextTick1, nextTick2`.

进入`check`阶段, 执行`setImmediate`的 callback, `输出 setImmediate1, setImmediate2`, 并在 callback 中设定了`process.nextTick()`的 callback.

结束`check`阶段, 进入`close callback`阶段之前, 执行`process.nextTick()`的 callback, `输出 nextTick3, nextTick4`.

最后进入`timer`阶段执行`setTimeout`的 callback, `输出setTimeout`

## 参考资料

- [event-loop-timers-and-nexttick](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

- [Node.js Event Loop 的理解 Timers，process.nextTick()](https://cnodejs.org/topic/57d68794cb6f605d360105bf)
