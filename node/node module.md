# node module

在 Node.js 模块系统中，每个文件都被视为独立的模块。模块内的本地变量是私有的，因为模块被 Node.js 包装在一个函数中（即模块包装器）。

### 模块包装器

在执行模块代码之前，Node.js 会使用一个如下的函数包装器将其包装

```js
(function(exports, require, module, __filename, __dirname) {
  // Module code actually lives in here
});
```

作用:

- 它保持了顶层的变量（用 var、const 或 let 定义）作用在模块范围内，而不是全局对象。
- 它有助于提供一些看似全局的但实际上是模块特定的变量，例如：
  - 实现者可以使用`module` 和 `exports` 对象从模块中导出值
  - 当前模块绝对路径文件名`__filename`和当前模块的文件夹名称 `__dirname`
    - 如在 `/Users/mjr` 目录下执行 `node example.js`
      ```js
      console.log(__filename);
      // Prints: /Users/mjr/example.js
      console.log(__dirname);
      // Prints: /Users/mjr
      ```

### `module.exports`和`exports`

`exports` 变量是在模块的文件级别作用域内有效的，它在模块被执行前被赋予 `module.exports` 的值。换句话说, `exports`是`module.exports`的快捷方式, 它们都指向同一个对象的引用.

然而, 当`exports`被赋予一个新的值时, 它将不再绑定到`module.exports`.

```js
module.exports.hello = true; // Exported from require of module
exports = { hello: false }; // Not exported, only available in the module
```

而当`module.exports`被赋予一个新的值时, 也会重新赋值`exports`, 类似于:

```js
module.exports = exports = function Constructor() {
  // ... etc.
};
```

`require`的类似实现:

注意, `require`返回的是`module.exports`而不是`exports`(所以当`exports`被赋予一个新的值时, 模块导出结果不会改变)

```js
function require(/* ... */) {
  const module = { exports: {} };
  ((module, exports) => {
    // Module code here. In this example, define a function.
    function someFunc() {}
    exports = someFunc;
    // At this point, exports is no longer a shortcut to module.exports, and
    // this module will still export an empty default object.
    module.exports = someFunc;
    // At this point, the module will now export someFunc, instead of the
    // default object.
  })(module, module.exports);
  return module.exports;
}
```

### 缓存

模块在第一次加载后会被缓存--把模块返回的数据(`module.exports`对象)保存到内存中。 这也意味着（类似其他缓存机制）如果每次调用 `require('foo')` 都解析到同一文件，则会返回同样的对象。

被引入的模块将被缓存在`require.cache`对象中。从该对象中删除键值(如`delete require.cache[require.resolve('./b.js')]`)将会导致下一次 `require` 时重新加载被删除的模块(注意不能删除原生插件(native addons))

多次调用 `require(foo)` 不会导致模块的代码被执行多次。 这是一个重要的特性。 借助它, 可以返回“部分完成”的对象(见下文的`循环`)，从而允许加载依赖的依赖, 即使它们会导致循环依赖。

如果想要多次执行一个模块，可以导出一个函数，然后调用该函数。

`注意`:

- 模块是基于其解析的文件名进行缓存的。 由于调用模块的位置的不同，模块可能被解析成不同的文件名（比如从 `node_modules` 目录加载），这样就不能保证 `require('foo')` 总能返回完全相同的对象(因为可能会被解析到不同的文件)。

- 此外，在不区分大小写的文件系统或操作系统中，被解析成不同的文件名可以指向同一文件，但缓存仍然会将它们视为不同的模块，并多次重新加载。 例如，`require('./foo')` 和 `require('./FOO')` 返回两个不同的对象，而不会管 `./foo`和 `./FOO` 是否是相同的文件。

### 循环

当循环调用 `require()` 时，一个模块可能在未完成执行时被返回。

```js
// a.js
console.log('a 开始');
exports.done = false;
// 引用 b 模块, b模块会执行完
const b = require('./b.js');
console.log('在 a 中，b.done = %j', b.done); // b.done = true
exports.done = true;
console.log('a 结束');

// b.js
console.log('b 开始');
exports.done = false;
// 由于a, b模块互相引用, 为了防止无限循环, 先返回a的未完成的副本, 此时 a 的exports.done = false;
const a = require('./a.js');
console.log('在 b 中，a.done = %j', a.done); // a.done = false
exports.done = true;
console.log('b 结束');

// main.js, 程序主入口
console.log('main 开始');
const a = require('./a.js');
// 引用 b 模块, 此时 b 模块已经在 a 模块中被require过, 模块代码不会重新执行
const b = require('./b.js');
console.log('在 main 中，a.done=%j，b.done=%j', a.done, b.done);
```

当 `main.js` 加载 `a.js` 时，`a.js` 又加载 `b.js`。 此时，`b.js` 会尝试去加载 `a.js`。 为了防止无限的循环，会返回一个 `a.js` 的 `exports` 对象的 **未完成的副本** 给 `b.js` 模块。 然后 `b.js` 完成加载，并将 `exports` 对象提供给 `a.js` 模块。

当 `main.js` 加载这两个模块时，它们都已经完成加载。因此，该程序的输出会是：

```js
$ node main.js
main 开始
a 开始
b 开始
在 b 中，a.done = false
b 结束
在 a 中，b.done = true
a 结束
在 main 中，a.done=true，b.done=true
```

### `require`的解析过程

`require(X) from module at path Y`

1.  `X` 如果是核心模块, 则返回该核心模块

2.  `X` 如果以`'/'`开头, 把 `Y` 设置为文件系统根目录

3.  `X` 如果以`'/'`,`'./'`, `'../'`开头

    - LOAD_AS_FILE(X) (X = Y + X)

      - 如果 `X` 是文件, 则 `X` 会被解析为 JavaScript 文本文件
      - 如果 `X.js` 是文件, 则 `X.js` 会被解析为 JavaScript 文本文件
      - 如果 `X.json` 是文件, 则 `X.json` 会被解析为 JavaScript Object
      - 如果 `X.node` 是文件, 则 `X.node` 会被解析为通过 `dlopen` 加载的编译后的插件模块

    - 否则 LOAD_AS_DIRECTORY(X) (X = Y + X)
      - 如果 `X/package.json` 是文件
        - 解析 `X/package.json` 并寻找 `main` 字段
        - let M = X + (json main field)
        - LOAD_AS_FILE(M)
        - 否则尝试加载 `M` 下的 `index.js`/`index.json`/`index.node`
      - 否则尝试加载 `X` 下的 `index.js`/`index.json`/`index.node`

4.  LOAD_NODE_MODULES(X, START) (START = dirname(Y))

    - 从当前模块(即`START`)的父目录开始, 尝试从它的`/node_modules`目录里加载模块(同时 Node.js 不会附加 `node_modules` 到一个已经以 `node_modules` 结尾的路径上):
      - LOAD_AS_FILE(X)
      - LOAD_AS_DIRECTORY(X)
    - 如果没有找到, 则移动到上一级父目录继续查找, 以此类推, 直到访问到当前项目路径的根目录. 如果此时仍未找到, 则报错.
    - 举个栗子, 如果在`'/home/ry/projects/foo.js'` 文件里调用了 `require('bar.js')`，则 Node.js 会按以下顺序查找:
      - `/home/ry/projects/node_modules/bar.js`
      - `/home/ry/node_modules/bar.js`
      - `/home/node_modules/bar.js`
      - `/node_modules/bar.js`

5.  如果以上均未找到, 则抛出异常`not found`

`注意`: 使用 `npm install -g xxx`安装了 xxx 模块时, 在项目中引用可能会提示找不到, 因为全局安装的模块通常是安装在其他路径, 而不在项目路径上. (官网称当在其他地方找不到模块时, Node.js 会搜索`NODE_PATH`环境变量里的路径, 但本地用 mac 测试的时候查找失败. 另外, 除了命令行工具使用到的模块, 一般模块不建议全局安装.)

```js
// require解析过程伪代码
require(X) from module at path Y
1. If X is a core module,
   a. return the core module
   b. STOP
2. If X begins with '/'
   a. set Y to be the filesystem root
3. If X begins with './' or '/' or '../'
   a. LOAD_AS_FILE(Y + X)
   b. LOAD_AS_DIRECTORY(Y + X)
4. LOAD_NODE_MODULES(X, dirname(Y))
5. THROW "not found"

LOAD_AS_FILE(X)
1. If X is a file, load X as JavaScript text.  STOP
2. If X.js is a file, load X.js as JavaScript text.  STOP
3. If X.json is a file, parse X.json to a JavaScript Object.  STOP
4. If X.node is a file, load X.node as binary addon.  STOP

LOAD_INDEX(X)
1. If X/index.js is a file, load X/index.js as JavaScript text.  STOP
2. If X/index.json is a file, parse X/index.json to a JavaScript object. STOP
3. If X/index.node is a file, load X/index.node as binary addon.  STOP

LOAD_AS_DIRECTORY(X)
1. If X/package.json is a file,
   a. Parse X/package.json, and look for "main" field.
   b. let M = X + (json main field)
   c. LOAD_AS_FILE(M)
   d. LOAD_INDEX(M)
2. LOAD_INDEX(X)

LOAD_NODE_MODULES(X, START)
1. let DIRS=NODE_MODULES_PATHS(START)
2. for each DIR in DIRS:
   a. LOAD_AS_FILE(DIR/X)
   b. LOAD_AS_DIRECTORY(DIR/X)

NODE_MODULES_PATHS(START)
1. let PARTS = path split(START)
2. let I = count of PARTS - 1
3. let DIRS = []
4. while I >= 0,
   a. if PARTS[I] = "node_modules" CONTINUE
   b. DIR = path join(PARTS[0 .. I] + "node_modules")
   c. DIRS = DIRS + DIR
   d. let I = I - 1
5. return DIRS
```

### 主模块

当 Node.js 直接运行一个文件时，`require.main` 会被设为它的 `module`。 这意味着可以通过 `require.main === module` 来判断一个文件是否被直接运行.

对于 `foo.js` 文件，如果通过 `node foo.js` 运行则为 true，但如果通过 `require('./foo')` 运行则为 false。

因为 `module` 提供了一个 `filename` 属性（通常等同于 `__filename`），所以可以通过检查 `require.main.filename` 来获取当前应用程序的入口点。

## 参考资料

- [node module(模块)](http://nodejs.cn/api/modules.html)
