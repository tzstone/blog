# Promise、async、await

## Promise

### 为什么要用 promises？

如果你读过关于 promises 的一些文章，你经常会发现对《世界末日的金字塔》这篇文章的引用，会有一些可怕的逐渐延伸到屏幕右侧的回调代码。

![callback hell](https://github.com/tzstone/MarkdownPhotos/raw/master/callback_hell.jpg)

promises 确实解决了这个问题，但它并不只是关乎于`缩进`。正如在优秀的演讲《回调地狱的救赎》中所解释的那样，`回调`真正的问题在于它们剥夺了我们对一些像 `return` 和 `throw` 的关键词的使用能力。相反，我们的程序的整个流程都会基于一些`副作用`。一个函数单纯的调用另一个函数。

事实上，回调做了很多更加险恶的事情：它们剥夺了我们的`堆栈`，这些是我们在编程语言中经常要考虑的。没有堆栈来书写代码在某种程度上就好比驾车没有刹车踏板：你不会知道你是多么需要它，直到你到达了却发现它并不在这。

回调的其他问题:

- 我们的大脑用`顺序`的，阻塞的，单线程的语义方式规划事情，但是回调使用非线性，非顺序的方式表达异步流程，这使我们正确推理这样的代码变得非常困难。
- `控制反转`: 回调需要你把你程序的一部分拿出来并把它执行的控制权移交给另一个第三方。这种控制权的转移使我们得到一张信任问题的令人不安的列表:
  - 回调重复调用/从不调用/调用太早/调用太晚
  - 没能传递必要的环境/参数
  - 吞掉了任何可能发生的错误/异常

promises 的意义在于它给回了在函数式语言里面我们遇到异步时所`丢失的 return，throw 和堆栈`。

另一方面, promise 能够 `反向倒转` 回调所面临的 `控制反转`(即`控制非反转`): 不是将我们的程序的延续包装进一个回调函数中，将这个回调交给另一个团体（甚至是潜在的外部代码），并双手合十祈祷它会做正确的事情并调用这个回调, 而是希望另一个团体返回给我们一个可以知道它何时完成的能力，然后我们的代码可以决定下一步做什么。[Promise 如何解决信任问题](https://www.bookstack.cn/read/You-Dont-Know-JS-async-performance/ch3.3.md)

`控制非反转` 导致了更好的关注分离:

```javascript
var evt = foo(42);
// 让`bar(..)`监听`foo(..)`的完成
bar(evt);
// 同时，让`baz(..)`监听`foo(..)`的完成
baz(evt);
```

也就是 bar(..)和 baz(..)不必卷入 foo(..)是如何被调用的问题。相似地，foo(..)也不必知道或关心 bar(..)和 baz(..)的存在或它们是否在等待 foo(..)完成的通知。

evt 事件监听能力是一个 Promise 的类比。在一个基于 Promise 的方式中，前面的代码段将会使 foo(..)创建并返回一个 Promise 实例，而且这个 promise 将会被传入 bar(..)和 baz(..)。

### Promise 的定义

根据 Promises/A+ 规范([英文版](https://promisesaplus.com/#point-1)), Promise 表示一个异步操作的最终结果，与之进行交互的方式主要是 then 方法，该方法注册了两个 `回调函数，用于接收 promise 的终值或本 promise 不能执行(reject)的原因(类似闭包)` 。以下是 [Promises/A+ 规范](https://www.ituring.com.cn/article/66566)的内容:

### Promise 的状态

一个 Promise 的当前状态必须为以下三种状态中的一种：等待态（Pending）、执行态（Fulfilled）和拒绝态（Rejected）。状态的流转图如下:

![promise status](https://github.com/tzstone/MarkdownPhotos/raw/master/promise_status.jpeg)

其中 promise 处于执行态（Fulfilled）和拒绝态（Rejected）时分别必须拥有一个 `不可变的终值` 和一个 `不可变的拒绝原因` 。这里的不可变指的是 `恒等` （即可用 `===` 判断相等），而不是意味着更深层次的不可变（指当 value 或 reason 不是基本值时，只要求其引用地址相等，但`属性值可被更改`）。

### Promise 的创建

注: 核心的 Promises/A+ 规范不涉及如何创建 fulfill 或 reject Promise，而是选择专注于提供可互操作的 then 方法。

```js
new Promise( function(resolve, reject) {...} /* executor */  )
```

- 构建 Promise 对象时，需要传入一个 executor 函数，主要业务流程都在 executor 函数中执行。
- Promise 构造函数执行时立即调用 executor 函数， resolve 和 reject 两个函数作为参数传递给 executor，resolve 和 reject 函数被调用时，分别将 promise 的状态改为 fulfilled（完成）或 rejected（失败）。一旦状态改变，就不会再变，任何时候都可以得到这个结果。

### then 方法

一个 promise 必须提供一个 then 方法以访问其当前值、终值和拒绝原因。promise 的 then 方法接受两个参数：

    promise.then(onFulfilled, onRejected)

- onFulfilled 特性

  - 如果 onFulfilled 不是函数，其必须被`忽略`
  - 如果 onFulfilled 是函数：
    - 当 promise 执行结束后其必须被调用，其`第一个参数`为 promise 的终值(其他参数则会被忽略)
    - 在 promise 执行结束前其不可被调用
    - 其调用次数不可超过一次

- onRejected 特性
  - 如果 onRejected 不是函数，其必须被`忽略`
  - 如果 onRejected 是函数：
    - 当 promise 被拒绝执行后其必须被调用，其`第一个参数`为 promise 的据因(其他参数则会被忽略)
    - 在 promise 被拒绝执行前其不可被调用
    - 其调用次数不可超过一次

onFulfilled 或 onRejected 只在执行环境堆栈只包含平台代码之后调用(确保 onFulfilled 和 onRejected 方法异步执行，且应该在 `then` 方法被调用的那一轮事件循环之后的新执行栈中执行)。

onFulfilled 和 onRejected 必须被作为函数调用（即没有 this 值）, 也就是说在严格模式（strict）中，函数 this 的值为 undefined ；在非严格模式中其为全局对象。

#### 多次调用

then 方法可以被同一个 promise `调用多次`

- 当 promise 成功执行时，所有 onFulfilled 需按照其`注册顺序`依次回调
- 当 promise 被拒绝执行时，所有的 onRejected 需按照其`注册顺序`依次回调

#### 返回

then 方法必须返回一个 promise 对象:

    promise2 = promise1.then(onFulfilled, onRejected);

- 如果 onFulfilled 或者 onRejected 返回一个值 x ，则运行下面的 Promise 解决过程：`[[Resolve]](promise2, x)`
- 如果 onFulfilled 或者 onRejected 抛出一个异常 e ，则 promise2 必须拒绝执行，并返回拒因 e
- 如果 onFulfilled `不是函数`且 promise1 成功执行， promise2 必须成功执行并返回`相同的值(promise1的值)`
- 如果 onRejected `不是函数` 且 promise1 拒绝执行， promise2 必须拒绝执行并返回 `相同的据因(promise1的拒因)`

不论 promise1 被 reject 还是被 resolve 时 promise2 都会被 resolve，只有出现异常时才会被 rejected。

### Promise 解决过程

Promise 解决过程是一个抽象的操作，其需输入一个 promise 和一个值，我们表示为 `[[Resolve]](promise, x)` ，如果 x 有 then 方法且看上去像一个 Promise ，解决程序即尝试使 promise 接受 x 的状态；否则其用 x 的值来执行 promise 。运行 `[[Resolve]](promise, x)` 需遵循以下步骤：

#### x 与 promise 相等

如果 promise 和 x 指向同一对象，以 TypeError 为据因拒绝执行 promise

#### x 为 Promise

如果 x 为 Promise ，则使 promise `接受 x 的状态`:

- 如果 x 处于等待态， promise 需保持为等待态直至 x 被执行或拒绝
- 如果 x 处于执行态，用相同的值执行 promise
- 如果 x 处于拒绝态，用相同的据因拒绝 promise

#### x 为对象或函数

1. 把 `x.then` 赋值给 `then`
2. 如果取 `x.then` 的值时抛出错误 `e` ，则以 `e` 为据因拒绝 promise
3. 如果 `then` 是函数，用 `x` 调用它(将 `then` 的 `this` 指向 `x` )。传递两个回调函数作为参数，第一个参数叫做 `resolvePromise` ，第二个参数叫做 `rejectPromise`:
   - 如果 `resolvePromise` 以值 `y` 为参数被调用，则运行 `[[Resolve]](promise, y)`
   - 如果 `rejectPromise` 以据因 `r` 为参数被调用，则以据因 `r` 拒绝 `promise`
   - 如果 resolvePromise 和 rejectPromise 均被调用，或者被同一参数调用了多次，则优先采用`首次调用`并忽略剩下的调用
   - 如果调用 `then` 方法抛出了`异常` e：
     - 如果 resolvePromise 或 rejectPromise 已经被调用，则忽略之
     - 否则以 e 为据因拒绝 promise
4. 如果 then `不是函数`，以 x 为参数执行 promise

#### x 不为对象或函数

以 x 为参数执行 promise（即 resolve(x)）

如果一个 promise 被一个循环的 thenable 链中的对象解决，而 [[Resolve]](promise, thenable) 的递归性质又使得其被再次调用，根据上述的算法将会陷入无限递归之中。算法虽不强制要求，但也鼓励施者检测这样的递归是否存在，若检测到存在则以一个可识别的 TypeError 为据因来拒绝 promise。

### API/进阶用法

#### Promise.prototype.catch()

返回一个 Promise，并且处理拒绝的情况。它的行为与调用 Promise.prototype.then(undefined, onRejected) 相同。然而，这并不意味着下面两个片段也是等价的：

```javascript
somePromise()
  .then(function () {
    return someOtherPromise();
  })
  .catch(function (err) {
    // handle error
  });

// 当你使用then(resolveHandler,rejectHandler)格式，如果resolveHandler自己抛出一个错误, rejectHandler并不能捕获。
somePromise().then(
  function () {
    return someOtherPromise();
  },
  function (err) {
    // handle error
  },
);
```

#### Promise.prototype.finally()

返回一个 Promise。在 promise 结束时，无论结果是 fulfilled 或者是 rejected，都会执行指定的回调函数。这为在 Promise 是否成功完成后都需要执行的代码提供了一种方式。

#### Promise.all()

接收一个 promise 的 iterable 类型（注：Array，Map，Set 都属于 ES6 的 iterable 类型）的输入，并且只返回一个 Promise 实例，那个输入的所有 promise 的 resolve 回调的结果是一个数组。

这个 Promise 的 resolve 回调执行是在所有输入的 promise 的 resolve 回调都结束，或者输入的 iterable 里没有 promise 了的时候。

它的 reject 回调执行是，只要任何一个输入的 promise 的 reject 回调执行或者输入不合法的 promise 就会立即抛出错误，并且 reject 的是第一个抛出的错误信息。

数组的所有项都 fulfill 时, promise fulfill, 数组任何一项 reject 时, promise reject。

如果一个空的 array 被传入 Promise.all([ .. ])，它会立即完成。

#### Promise.allSettled()

返回一个在所有给定的 promise 都已经 fulfilled 或 rejected 后的 promise，并带有一个对象数组，每个对象表示对应的 promise 结果。跟 Promise.all() 类似, 但不会进行短路

注: 有兼容性问题

#### Promise.any()

Promise.any() 接收一个 Promise 可迭代对象，只要其中的一个 promise 成功，就返回那个已经成功的 promise 。如果可迭代对象中没有一个 promise 成功（即所有的 promises 都失败/拒绝），就返回一个失败的 promise 和 AggregateError 类型的实例，它是 Error 的一个子类，用于把单一的错误集合在一起。本质上，这个方法和 Promise.all()是相反的。

注: 有兼容性问题

#### Promise.race(iterable)

返回一个 promise，一旦迭代器中的某个 promise 解决或拒绝，返回的 promise 就会解决或拒绝。

注意： 一个“竞合（race）”需要至少一个“选手”，所以如果你传入一个空的 array，race([..])的主 Promise 将不会立即解析，反而是永远不会被解析。

#### Promise.resolve(value)

返回一个以给定值解析后的 Promise 对象。如果这个值是一个 promise ，那么将返回这个 promise ；如果这个值是 thenable（即带有"then" 方法），返回的 promise 会“跟随”这个 thenable 的对象，采用它的最终状态；否则返回的 promise 将以此值完成。此函数将类 promise 对象的多层嵌套展平。

该方法配合 .catch() 方法可以方便地捕获任意的同步错误。

```js
function somePromiseAPI() {
  return Promise.resolve()
    .then(function () {
      doSomethingThatMayThrow();
      return 'foo';
    })
    .then(/* ... */)
    .catch(/* ... */);
}
```

#### promise 工厂

一个接一个的，在一个序列中执行一系列的 promise

```js
// Loop through our chapter urls
story.chapterUrls.reduce(function (sequence, chapterUrl) {
  // Add these actions to the end of the sequence
  return sequence
    .then(function () {
      return getJSON(chapterUrl);
    })
    .then(function (chapter) {
      addHtmlToPage(chapter.html);
    });
}, Promise.resolve());

// 另一种实现
function executeSequentially(promiseFactories) {
  var result = Promise.resolve();
  promiseFactories.forEach(function (promiseFactory) {
    result = result.then(promiseFactory);
  });
  return result;
}
```

### 反模式

#### 世界末日的 promise 金字塔

```javascript
remotedb.allDocs({
  include_docs: true,
  attachments: true
}).then(function (result) {
  var docs = result.rows;
  docs.forEach(function(element) {
    localdb.put(element.doc).then(function(response) {
      alert("Pulled doc with id " + element.doc._id + " and added to local db.");
    }).catch(function (err) {
      if (err.status == 409) {
        localdb.get(element.doc._id).then(function (resp) {
          localdb.remove(resp._id, resp._rev).then(function (resp) {
// et cetera...
```

更好的方式: 组成式 promises（composing promises）。每一个函数都在前面的 promise 被 resolve 之后被调用，而且将前面的 promise 的输出作为参数被调用。

```javascript
// good
remotedb.allDocs(...).then(function (resultOfAllDocs) {
  return localdb.put(...);
}).then(function (resultOfPut) {
  return localdb.get(...);
}).then(function (resultOfGet) {
  return localdb.put(...);
}).catch(function (err) {
  console.log(err);
});
```

#### 错误使用 forEach()

```javascript
// I want to remove() all docs
db.allDocs({ include_docs: true })
  .then(function (result) {
    result.rows.forEach(function (row) {
      db.remove(row.doc);
    });
  })
  .then(function () {
    // I naively believe all docs have been removed() now!
  });
```

这些代码有什么问题呢？问题在于第一个函数返回 undefined，意味着第二个函数并不是在等待 db.remove()在所有文件上被调用。实际上，它没有在等任何东西，并且在任何数量的文件被删除的时候都可能会执行。

```js
// good
db.allDocs({ include_docs: true })
  .then(function (result) {
    return Promise.all(
      result.rows.map(function (row) {
        return db.remove(row.doc);
      }),
    );
  })
  .then(function (arrayOfResults) {
    // All docs have really been removed() now!
  });
```

#### 不添加 .catch()

许多开发者会很自豪的认为他们的 promises 代码永远都不会出错，于是他们忘记在代码中添加.catch()方法。不幸的是，这会导致任何被抛出的错误都会被吞噬掉，甚至在你的控制台你也不会发现有错误输出。这在 debug 代码的时候真的会非常痛苦。

```js
// good
somePromise()
  .then(function () {
    return anotherPromise();
  })
  .then(function () {
    return yetAnotherPromise();
  })
  .catch(console.log.bind(console)); // <-- this is badass
```

#### 不使用 return

```js
somePromise()
  .then(function () {
    someOtherPromise();
  })
  .then(function () {
    // Gee, I hope someOtherPromise() has resolved!
    // Spoiler alert: it hasn't.
  });
```

第一个 `then` 回调中没有显式 `return`, 因此函数默认返回了一个同步值 `undefined`, 这意味着第二个 `then` 回调其实没有等到 someOtherPromise()被 resolve 就会被调用。

好的方式是 then() 函数里总是返回数据(promise 或同步值)或者抛出异常。

```js
// good
getUserByName('nolan')
  .then(function (user) {
    return getUserAccountById(user.id);
  })
  .then(function (userAccount) {
    // I got a user account!
  });
```

#### promise 丢失

```javascript
Promise.resolve('foo')
  .then(Promise.resolve('bar'))
  .then(function (result) {
    console.log(result);
  });
```

以上代码上会打印出 foo。原因是当你给 then() 传递一个非函数（比如一个 promise）值的时候，它实际上会解释为 then(null)，这会导致之前的 promise 的结果丢失。所以永远记得给 then() 传递一个函数参数。

## async/await

参考资料:

[Promises/A+ 规范](https://www.ituring.com.cn/article/66566)

[Promise -- MDN](https://developer.cdn.mozilla.net/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)

[JavaScript Promises: An introduction](https://web.dev/promises/)

[promises 很酷，但很多人并没有理解就在用了](https://mp.weixin.qq.com/s?__biz=MzAxODE2MjM1MA==&mid=2651551800&idx=1&sn=d06d319c002fdca153bc2abe9352e959&chksm=8025aff9b75226efe21a5094ce14a29c467be74ef2eb631157ea2732106642357617935e9464&mpshare=1&scene=23&srcid=0327MmTcVFfM68Q1mgoCCZ1Y#rd)

[你不懂 JS: 异步与性能（You Dont Know JS）-- Promise](https://www.bookstack.cn/read/You-Dont-Know-JS-async-performance/ch3.md)

[你真的懂 Promise 吗](https://blog.csdn.net/howgod/article/details/105628598)

https://developers.google.com/web/fundamentals/primers/async-functions

[理解 JavaScript 的 async/await](https://segmentfault.com/a/1190000007535316)

[[译]驯服 ES7 的异步野兽 - async](https://zhuanlan.zhihu.com/p/25652957?from_voters_page=true)
