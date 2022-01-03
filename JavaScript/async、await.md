# async、await

async 函数是使用 async 关键字声明的函数。 async 函数是 AsyncFunction 构造函数的实例，并且其中允许使用 await 关键字。await 关键字只在 async 函数内有效。如果你在 async 函数体之外使用它，就会抛出语法错误 SyntaxError 。

## await

```code
[返回值] = await 表达式;
```

`表达式`: 可以是一个 Promise 对象或者任何要等待的`值`。

`返回值`: 返回 Promise 对象的`处理结果`。如果等待的不是 Promise 对象，则返回该`值本身`(将该值转换为已正常处理的 Promise，然后等待其处理结果)。

`await` 表达式会 `暂停` 当前 `async` 函数的 `执行进程` 并出让其 `控制权`，只有当其等待的基于 promise 的异步操作 `被解决(funfilled)` 或 `被拒绝(rejected)` 之后才会 `恢复进程`。若 Promise 正常处理(fulfilled)，其回调的 resolve 函数参数作为 await 表达式的 `值`。若 Promise 处理异常(rejected)，await 表达式会把 Promise 的异常原因抛出。

注意: await 只会`暂停`它的`父函数`, 所以在循环中使用必须小心:

```javascript
const plist = [p1, p2, p3]
// p1, p2, p3 串行发起
async function func() {
  for (let i = 0; i < plist.length; i++) {
    // await暂停整个func函数
    await plist[i]()
  }
}
// p1, p2, p3 并行发起
plist.forEach(async function (p) => {
  // await暂停当前async函数
  await p()
})
```

在 `await 表达式之后的代码`可以被认为是存在于链式调用的 `then 回调`中，多个 await 表达式都将加入链式调用的 then 回调中，`返回值`将作为`最后一个 then 回调`的返回值。

## async

async 函数的函数体可以被看作是由 0 个或者多个 await 表达式分割开来的。从第一行代码直到（并包括）第一个 await 表达式（如果有的话）都是`同步运行`的。这样的话，一个`不含 await 表达式`的 async 函数是会`同步运行`的。然而，如果函数体内有一个 `await` 表达式，`async` 函数就一定会`异步执行`。

```javascript
async function foo() {
  await 1;
}
// 等价于
function foo() {
  // await后面没有代码, 函数默认返回undefined
  return Promise.resolve(1).then(() => undefined);
}
```

使用 async/await 关键字就可以在`异步代码`中使用普通的 `try/catch` 代码块。`try/catch` 往往是必须的, 否则当 promise 返回 `reject` 的时候，错误不会被抛出，因此你看不到代码的问题所在。

## 返回

async 函数返回一个 `Promise`，这个 promise 要么会通过一个由 async 函数返回的值`被解决`，要么会通过一个从 async 函数中抛出的（或其中没有被捕获到的）异常`被拒绝`。

`async` 函数一定会返回一个 `promise` 对象。如果一个 async 函数的返回值看起来不是 promise，那么它将会被`隐式`地包装在一个 promise 中。

```javascript
async function foo() {
  await 1;
}
// 等价于
function foo() {
  return Promise.resolve(1).then(() => undefined);
}
```

### return await promiseValue 与 return promiseValue

返回值隐式的传递给 Promise.resolve，并不意味着 return await promiseValue 和 return promiseValue 在功能上相同。

```js
async function func(v) {
  try {
    // foo(v) 为一个promise对象
    return await foo(v);
  } catch (e) {
    return null;
  }
}
```

return foo 和 return await foo 有一些细微的差异: return foo; 不管 foo 是 promise 还是 rejects 都将会直接返回 foo。相反地，如果 foo 是一个 Promise，return await foo; 将等待 foo 执行(resolve)或拒绝(reject)，如果是拒绝，将会在返回前抛出异常, 进而返回 null。

## 与 Promise 相比

- 代码上更接近于同步代码, 更清晰
- 在多个顺序执行的 promise 中传参更为方便(假使每一个 promise 都需要之前所有 promise 的结果)

参考资料:

[async 函数 -- MDN](https://developer.mozilla.org/zh-cn/docs/web/javascript/reference/statements/async_function)

[await -- MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/await)

[[译]驯服 ES7 的异步野兽 - async](https://zhuanlan.zhihu.com/p/25652957?from_voters_page=true)

[Async functions - making promises friendly](https://developers.google.com/web/fundamentals/primers/async-functions)

[理解 JavaScript 的 async/await](https://segmentfault.com/a/1190000007535316)
