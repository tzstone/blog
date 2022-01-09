# javaScript 动态代码安全隔离

最近做了一个需求, 需要在页面上给用户提供编写 js 代码的功能, 用来完成一些动态参数转换. 这种有动态执行能力的代码无法进行常规的静态代码分析(如 sonar), 往往有很大的安全隐患, 比如 XSS. 因此需要将这种动态 js 代码的执行隔离起来, 限制其对宿主环境的访问(目前的使用场景无需访问宿主环境)并尽可能不影响到主线程(比如用 while 循环将主线程卡死).

有以下几种方式:

### web worker

web worker 独立于主线程, 不能访问宿主对象(DOM/BOM). 兼容 IE10.

```javascript
// 工作线程代码
function workCode(data) {
  onmessage = (e) => {
    const { args, func } = e.data;
    console.log("before", args);
    const result = new Function(`return ${func}`)().apply(
      null,
      JSON.parse(args)
    );
    postMessage(result);
    // 销毁
    close();
  };
}

const workBlob = new Blob([`(${workCode.toString()})()`]);
const url = URL.createObjectURL(workBlob);
const worker = new Worker(url);
// 数据
const data = [1, 2, 3];
// 待执行函数体
const func = `function(data){return data.map(t=> t+10)}`;
// 主线程向工作线程发送待执行函数及数据
worker.postMessage({
  args: JSON.stringify([data]), // 数据传递默认使用结构化克隆, 数据量大时用JSON.stringify性能更好
  func,
});
// 接收工作线程执行后的数据
worker.onmessage = (e) => {
  console.log("after:", e.data);
  // 销毁worker
  worker.terminate();
};
```

### proxy

通过代理 window 对象, 限制用户代码对 window 的访问, 不兼容 IE:

```javascript
// 以下属性在 with 执行环境不会被直接读取
var unscopables = {
  undefined: true,
  Array: true,
  Object: true,
  String: true,
  Boolean: true,
  Math: true,
  Number: true,
  Symbol: true,
  parseFloat: true,
  Float32Array: true,
};
var proxy = new Proxy(window, {
  get: function (target, prop, receiver) {
    if (prop === Symbol.unscopables) return unscopables; // https://www.zhihu.com/question/364970876
    if (prop === "alert") return "alert is not available";
    return window[prop];
  },
});
(function (window, self, globalThis) {
  with (window) {
    console.log(alert);
  }
}.bind(proxy)(proxy, proxy, proxy));
```

### with 语句

with 语句可以在作用域链的前端添加一个变量对象, 用户编写的 js 代码会先在该变量对象中查找, 其作用类似于 proxy.

```javascript
// 重写 alert 方法
with ({ alert: null }) {
  // 用户编写的js代码
  alert("test");
}
```

### 语法检查

在运行时对用户代码进行语法分析(AST), 找出其中不安全的语法, 如[js-x-ray](https://github.com/NodeSecure/js-x-ray).
