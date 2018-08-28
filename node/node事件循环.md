# node 事件循环

虽然 JavaScript 是单线程的, 但是通过尽可能将操作转移(offloading)到系统内核, 事件循环(event loop)允许 Node.js 执行非阻塞 I/O 操作.

由于大多数现代内核都是多线程的，因此它们可以处理在后台执行的多个操作。当其中一个操作完成时，内核会通知 Node.js，以便可以将相应的回调添加到轮询队列中以最终执行。

**事件循环顺序**:
![](https://github.com/tzstone/MarkdownPhotos/blob/master/node-event-loop.jpeg)

每个方框对应 事件循环 的一个阶段.
每个阶段有一个 FIFO 队列, 用来存放待执行的回调. 当 事件循环 进入某个给定的阶段时, 会执行该阶段的一些特定操作, 然后执行该阶段队列里的回调, 直到队列耗尽或者执行的回调数量达到最大值, 此时 事件循环 将移到下一阶段, 以此类推.

- `timers`: this phase executes callbacks scheduled by setTimeout() and setInterval().
  从技术上讲，poll 阶段控制何时执行定时器。
- `pending callbacks`: 执行某些系统操作的回调(如 TCP 错误)
- `idle, prepare`: only used internally.
- `poll`: 有两个重要的功能:

  - 计算它应该堵塞和轮询 I/O 的时间
  - 处理轮询队列中的事件

  当事件循环进入 poll 阶段并且没有 timer 任务时,

  - poll 队列非空: 事件循环将遍历执行(同步)poll 队列中的回调, 直到遍历完成或者达到系统相关的限制.
  - poll 队列为空:
    - 如果脚本已经调用了`setImmediate()`，则事件循环将结束 poll 阶段并进入 check 阶段, 以执行这些调度脚本。
    - 如果脚本没有调用过`setImmediate()`，则事件循环将等待回调被添加到队列，然后立即执行它们。

  一旦 poll 队列为空，事件循环 将检查已达到时间阈值的计时器。如果一个或多个计时器准备就绪，事件循环将回绕到 `timers` 阶段以执行那些计时器的回调。

- `check`: 此阶段允许人员在 poll 阶段完成后立即执行回调。
  通常，在执行代码时，事件循环 最终会到达 poll 阶段，它将等待传入连接(incoming connections)，请求等。但是，如果某个回调已经调用`setImmediate()`并且 poll 阶段变为空闲，则将结束 poll 并进入 check 阶段，而不是等待 poll 事件。
- `close callbacks`: 如果 socket 或者 handle 突然关闭(例如 socket.destroy())，则将在此阶段触发'close'事件。否则它将通过`process.nextTick()触发。

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

在 I/O 周期内运行, setImmediate 的回调通常会先执行

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

使用`setImmediate()`而不是`setTimeout()`的主要优点是: 如果在 I/O 周期内调度, `setImmediate()`将始终在任何定时器之前执行，不管当前存在多少定时器。

## `process.nextTick()`

从技术上讲, `process.nextTick()`不是事件循环的一部分。相反，`nextTickQueue`将在当前操作完成后处理，而不管事件循环的当前阶段如何。

在给定的阶段调用`process.nextTick()`时, 所有`process.nextTick()`的回调将会在当前的事件循环中执行. 这很容易造成问题--当递归调用`process.nextTick()`时, 事件循环无法进入 poll 阶段, I/O 会被堵塞.

## 参考资料

- [event-loop-timers-and-nexttick](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
