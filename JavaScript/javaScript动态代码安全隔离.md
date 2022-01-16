# javaScript 动态代码安全隔离

最近做了一个需求, 需要在页面上给用户提供编写 js 代码的功能, 用来完成一些动态参数转换. 这种有动态执行能力的代码无法进行常规的静态代码分析(如 sonar), 往往有很大的安全隐患, 比如 XSS. 因此需要将这种动态 js 代码的执行隔离起来, 限制其对宿主环境的访问(目前的使用场景无需访问宿主环境)并尽可能不影响到主线程(比如用 while 循环将主线程卡死).

有以下几种方式:

### web worker

优点: 独立于主线程的单独线程, 工作线程即使陷入死循环也不影响主线程.

缺点: 不能访问宿主对象(DOM/BOM); 只兼容到 IE10; 处理结果回传需要进行异步处理.

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
const func = `function(data){
  const proxy = {
    data: data,
    navigator: null, 
    performance: null
    fetch: null,
    WebSocket: null,
    XMLHttpRequest: null, 
    setInterval: null, 
    setTimeout: null,
    Cache: null,
    CacheStorage: null,
    indexedDB: null
    localStorage: null,
    sessionStorage: null,
    CustomEvent: null,
    BarcodeDetector: null,
    BroadcastChannel: null,
    MessageChannel: null,
    Notification: null,
    EventSource: null
  }
  // 限制可访问对象
  with(proxy) {
    return data.map(t=> t+10)
  }
}`;
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
// 防止worker中执行了死循环代码
setTimeout(() => {
  worker.terminate();
}, 300);
```

### iframe

通过设置 data URI 类型的 src 属性, 可以使 iframe 与父容器跨域, 没有脚本注入问题, 也可以设置 sandbox 属性.

优点: 可以提供 sandbox 环境, 跨域 iframe 没有脚本注入问题; 兼容性好.

缺点: iframe 与父容器处于同一个线程, iframe 中的死循环会影响父容器代码执行; 处理结果可能涉及跨域 iframe 消息回传.

```javascript
var html = "<script>" + injected_script + "</script>";
var html_src = "data:text/html;charset=utf-8," + encodeURI(html);
iframe.src = html_src;
```

### proxy

通过代理 window 对象, 限制用户代码对 window 的访问, 可结合 with 语句限制对 with 对象的访问.

优点: 同步代码处理比较简单.

缺点: 不兼容 IE; 不完整的隔离环境, 执行代码可能影响主流程(如死循环代码); 隔离对象设置不如单独使用 with 简单.

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
    if (prop === "alert") throw new Error("alert is not available");
    return window[prop];
  },
});
(function (window, self, globalThis) {
  with (window) {
    console.log(alert);
  }
}.bind(proxy)(proxy, proxy, proxy));
```

### Object.defineProperty()

类似于 proxy, 可结合 with 语句限制对 with 对象的访问.

优点: 同步代码处理简单, 兼容性好(IE9).

缺点: 需遍历对象的每个属性进行设置, 无法像 proxy 那样对整个对象进行拦截; 不完整的隔离环境, 执行代码可能影响主流程(如死循环代码).

### with 语句

with 语句可以在作用域链的前端添加一个变量对象, 用户编写的 js 代码会先在该变量对象中查找, 其作用类似于 proxy.

优点: 同步代码处理简单; 兼容性好; 隔离对象设置相比 proxy 来得简单.

缺点: 使用 with 语句浏览器无法做编译上的优化; 不完整的隔离环境, 执行代码可能影响主流程(如死循环代码).

```javascript
// 重写 alert 方法
with ({ alert: null }) {
  // 用户编写的js代码
  alert("test");
}
```

### 语法分析

在运行时对用户代码进行语法分析(AST), 找出其中不安全的语法, 如[js-x-ray](https://github.com/NodeSecure/js-x-ray).

可以分析并修改用户代码, 如插入死循环代码检测退出代码.

优点: 可直接修改用户代码, 灵活性高; 可利用现有相对成熟的检测工具(npm 包), 对需向用户开放较大功能权限的代码检测比较方便.

缺点: 无法提供隔离环境, 只能进行静态语法分析检查, 对于动态的不安全代码检测能力有限; 需要对 AST 有一点了解.

死循环检测退出代码:

```javascript
import { parse } from "@babel/parser";
const code = `function abc() {
 while (true) {
  console.log(Date.now())
 }
}`;
const ast = parse(code);

// 死循环检测
const prefix = `
 let __count = 0
 const __detectInfiniteLoop = () => {
  if (__count > 10000) {
   throw new Error('Infinite Loop detected')
  }
  __count += 1
 }
`;
const detector = parse(`__detectInfiniteLoop()`);

// 在while/for/do..while语句中插入检测代码
import traverse from "@babel/traverse";
traverse(ast, {
  ForStatement: function (path) {
    path.node.body.body.push(...detector.program.body);
  },
  WhileStatement: function (path) {
    path.node.body.body.push(...detector.program.body);
  },
  DoWhileStatement: function (path) {
    path.node.body.body.push(...detector.program.body);
  },
});

// 生成最终代码
import generate from "@babel/generator";
const newCode = prefix + generate(ast).code;
```

参考资料:

[如何检测 JavaScript 中的死循环示例详解](https://www.jb51.net/article/194474.htm)

[为 Iframe 注入脚本的不同方式比较](https://harttle.land/2016/04/14/iframe-script-injection.html)
