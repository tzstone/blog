# nextTick

## JS 运行机制

JS 执行是单线程的，它是基于事件循环的。事件循环大致分为以下几个步骤：

（1）所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。

（2）主线程之外，还存在一个"任务队列"（task queue）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。

（3）一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。

（4）主线程不断重复上面的第三步。

<img src="https://ustbhuangyi.github.io/vue-analysis/assets/event-loop.png">

主线程的执行过程就是一个 tick，而所有的异步结果都是通过 “任务队列” 来调度。 消息队列中存放的是一个个的任务（task）。 规范中规定 task 分为两大类，分别是 macro task 和 micro task，并且每个 macro task 结束后，都要清空所有的 micro task。

## 流程梳理

- macroTimerFunc 和 microTimerFunc 的初始化
  - macroTimerFunc 检测顺序: 原生 setImmediate->原生 MessageChannel->setTimeout 0
  - microTimerFunc 检测顺序: 原生 Promise->macroTimerFunc

调用 nextTick 的时候会将回调函数 cb 推进 callbacks 队列中, 同时使用 pending 标志位来确保一个 tick 中多次调用 nextTick 只开启一个异步任务(根据 useMacroTask 执行 macroTimerFunc 或者 microTimerFunc. 默认走 microTimerFunc, 但在需要时(如 v-on 绑定的事件回调)会强制走 macroTimerFunc). 异步任务中会执行 flushCallbacks, 将 callbacks (由调用 nextTick 时传入的 cb 组成)队列挨个执行.

## 源码解读

```javascript
/* @flow */
/* globals MessageChannel */

import { noop } from 'shared/util';
import { handleError } from './error';
import { isIOS, isNative } from './env';

const callbacks = [];
let pending = false;

// 遍历执行回调
function flushCallbacks() {
  // 重置标志位
  pending = false;
  // 拷贝并清空callbacks, 这样如果执行回调的过程中调用了nextTick则会将cb推进callbacks队列, 不会在当前的flushCallbacks中执行, 而是进入到下一个异步任务中
  const copies = callbacks.slice(0);
  callbacks.length = 0;
  for (let i = 0; i < copies.length; i++) {
    copies[i]();
  }
}

// Here we have async deferring wrappers using both micro and macro tasks.
// In < 2.4 we used micro tasks everywhere, but there are some scenarios where
// micro tasks have too high a priority and fires in between supposedly
// sequential events (e.g. #4521, #6690) or even between bubbling of the same
// event (#6566). However, using macro tasks everywhere also has subtle problems
// when state is changed right before repaint (e.g. #6813, out-in transitions).
// Here we use micro task by default, but expose a way to force macro task when
// needed (e.g. in event handlers attached by v-on).
let microTimerFunc;
let macroTimerFunc;
let useMacroTask = false;

// Determine (macro) Task defer implementation.
// Technically setImmediate should be the ideal choice, but it's only available
// in IE. The only polyfill that consistently queues the callback after all DOM
// events triggered in the same loop is by using MessageChannel.
/* istanbul ignore if */
// macroTimerFunc检测顺序: 原生setImmediate->原生MessageChannel->setTimeout 0
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks);
  };
} else if (
  typeof MessageChannel !== 'undefined' &&
  (isNative(MessageChannel) ||
    // PhantomJS
    MessageChannel.toString() === '[object MessageChannelConstructor]')
) {
  const channel = new MessageChannel();
  const port = channel.port2;
  channel.port1.onmessage = flushCallbacks;
  macroTimerFunc = () => {
    port.postMessage(1);
  };
} else {
  /* istanbul ignore next */
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0);
  };
}

// Determine MicroTask defer implementation.
/* istanbul ignore next, $flow-disable-line */
// microTimerFunc检测顺序: 原生Promise->macroTimerFunc
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve();
  microTimerFunc = () => {
    p.then(flushCallbacks);
    // in problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop);
  };
} else {
  // fallback to macro
  microTimerFunc = macroTimerFunc;
}

/**
 * Wrap a function so that if any code inside triggers state change,
 * the changes are queued using a Task instead of a MicroTask.
 */
export function withMacroTask(fn: Function): Function {
  return (
    fn._withTask ||
    (fn._withTask = function () {
      useMacroTask = true;
      const res = fn.apply(null, arguments);
      useMacroTask = false;
      return res;
    })
  );
}

export function nextTick(cb?: Function, ctx?: Object) {
  let _resolve;
  // 将cb推进callbacks
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx);
      } catch (e) {
        handleError(e, ctx, 'nextTick');
      }
    } else if (_resolve) {
      // resolve()时进入Promise化调用的then方法里, 如nextTick().then(() => {})
      _resolve(ctx);
    }
  });
  // 保证同一个tick中多次调用nextTick不会开启多个异步任务, 而是在下一个tick执行
  if (!pending) {
    pending = true;
    // 默认走microTimerFunc, 对v-on绑定的事件回调会强制走macroTimerFunc
    if (useMacroTask) {
      macroTimerFunc();
    } else {
      microTimerFunc();
    }
  }
  // $flow-disable-line
  // 当调用nextTick不传cb时提供一个Promise化的调用, 如nextTick().then(() => {})
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise((resolve) => {
      _resolve = resolve;
    });
  }
}
```

```javascript
// DOM事件包装
function add(event: string, handler: Function, capture: boolean, passive: boolean) {
  handler = withMacroTask(handler);
  target.addEventListener(event, handler, supportsPassive ? { capture, passive } : capture);
}

function updateDOMListeners(oldVnode: VNodeWithData, vnode: VNodeWithData) {
  if (isUndef(oldVnode.data.on) && isUndef(vnode.data.on)) {
    return;
  }
  const on = vnode.data.on || {};
  const oldOn = oldVnode.data.on || {};
  target = vnode.elm;
  normalizeEvents(on);
  updateListeners(on, oldOn, add, remove, createOnceHandler, vnode.context);
  target = undefined;
}
```

参考资料:

[异步更新队列](https://cn.vuejs.org/v2/guide/reactivity.html#%E5%BC%82%E6%AD%A5%E6%9B%B4%E6%96%B0%E9%98%9F%E5%88%97)

[nextTick](https://ustbhuangyi.github.io/vue-analysis/v2/reactive/next-tick.html#js-%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6)

[Vue 进阶面试必问，异步更新机制和 nextTick 原理](https://mp.weixin.qq.com/s/ihzymIiY_i4Hj21CDHgzZQ)
