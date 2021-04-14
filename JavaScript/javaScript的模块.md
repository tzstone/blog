# JavaScript的模块

## 规范

### CommonJS

`CommonJS`规范采用`同步阻塞`的方式加载模块, 应用于`服务器端`(`nodejs`的模块系统就实现了CommonJS规范)。因为在服务器端，模块文件都存在`本地磁盘`，加载起来比较快，所以采用同步阻塞加载的方式不会有问题。但是在浏览器端，模块是存放在服务器端, 等待时间取决于网速的快慢, 等待时间过长会导致浏览器处于假死状态, 因此更合理的方案是使用异步加载。所以`浏览器端`一般采用`AMD/CMD`规范。

#### 模块定义

CommonJS规范规定，每个模块内部，`module`变量代表当前模块。这个变量是一个对象，它的`exports`属性（即`module.exports`）是对外的接口。加载某个模块，其实是加载该模块的`module.exports`属性。

```javascript
// example.js
var x = 5;
var addX = function (value) {
  return value + x;
};
// 输出变量x和函数addX
module.exports.x = x;
module.exports.addX = addX;
```

对外输出模块接口时, 也可以采用以下简写方式:

```javascript
var x = 5;
exports.x = x;
```

`module.exports`和`exports`指向同一个对象引用, 最终使用的仍然是`module.exports`去导出. 即:

```javascript
// 伪代码
var exports = {};  
var module = {  
    exports: exports  
}  
return module.exports  
```

注意不能直接将`exports`指向一个值, 这样会切断`exports`和`module.exports`的联系:

```javascript
exports = function(x) {console.log(x)}; // 这种写法是无效的
```

#### 模块加载

`require`方法用于加载模块(读入并执行一个JavaScript文件，然后返回该模块的`exports`对象):

```javascript
var example = require('./example.js'); // 执行到此时，example.js 同步下载并执行

console.log(example.x); // 5
console.log(example.addX(1)); // 6
```

CommonJS模块的加载机制是，输入的是被输出的值的`浅拷贝`。也就是说，一旦输出一个值，模块内部的变化就影响不到这个值(引用类型的值还是会影响)。

```javascript
// lib.js
var counter = 3;
function incCounter() {
  counter++;
}
module.exports = {
  counter: counter,
  incCounter: incCounter,
};

// main.js
var counter = require('./lib').counter;
var incCounter = require('./lib').incCounter;

console.log(counter);  // 3
incCounter();
console.log(counter); // 3
```

##### 模块加载规则(Node)

`require`命令用于加载文件，后缀名默认为`.js`

根据参数的不同格式，require命令去不同路径寻找模块文件。

1. 如果参数字符串以“`/`”开头，则表示加载的是一个位于绝对路径的模块文件。比如，`require('/home/marco/foo.js')`将加载`/home/marco/foo.js`。
2. 如果参数字符串以“`./`”开头，则表示加载的是一个位于相对路径（跟当前执行脚本的位置相比）的模块文件。比如，`require('./circle')`将加载当前脚本同一目录的`circle.js`。
3. 如果参数字符串不以“./“或”/“开头，则表示加载的是一个默认提供的`核心模块`（位于Node的系统安装目录中），或者一个位于各级`node_modules`目录的已安装模块（全局安装或局部安装）。

    举例来说，脚本`/home/user/projects/foo.js`执行了`require('bar.js')`命令，Node会依次搜索以下文件。

    ```code
    /usr/local/lib/node/bar.js
    /home/user/projects/node_modules/bar.js
    /home/user/node_modules/bar.js
    /home/node_modules/bar.js
    /node_modules/bar.js
    ```
4. 如果参数字符串不以“./“或”/“开头，而且是一个路径，比如require('example-module/path/to/file')，则将先找到example-module的位置，然后再以它为参数，找到后续路径。
5. 如果指定的模块文件没有发现，Node会尝试为文件名添加`.js`、`.json`、`.node`后，再去搜索。`.js`件会以文本格式的JavaScript脚本文件解析，`.json`文件会以JSON格式的文本文件解析，`.node`文件会以编译后的二进制文件解析。
6. 如果想得到require命令加载的确切文件名，使用`require.resolve()`方法。
7. 当require的参数字符串指向一个目录时，会自动查看该目录的`package.json`文件，然后加载`main`字段指定的入口文件。如果package.json文件没有main字段，或者根本就没有package.json文件，则会加载该目录下的`index.js`文件或`index.node`文件。

###### 环境变量NODE_PATH

Node执行一个脚本时，会先查看环境变量`NODE_PATH`。它是一组以冒号分隔的绝对路径。在其他位置找不到指定模块时，Node会去这些路径查找。

可以将`NODE_PATH`添加到`.bashrc`。

```javascript
export NODE_PATH="/usr/local/lib/node"
```

如果遇到复杂的相对路径，比如下面这样:

```javascript
var myModule = require('../../../../lib/myModule');
```
可以修改`NODE_PATH`环境变量，`package.json`文件可以采用下面的写法:

```javascript
{
  "main": "index.js",
  "scripts": {
    "start": "NODE_PATH=lib node index.js"
  },
}
```

##### require的处理流程

require命令是CommonJS规范之中，用来加载其他模块的命令。它其实不是一个全局命令，而是指向当前模块的module.require命令，而后者又调用Node的内部命令`Module._load`。

```javascript
Module._load = function(request, parent, isMain) {
  // 1. 检查 Module._cache，是否缓存之中有指定模块
  // 2. 如果缓存之中没有，就创建一个新的Module实例
  // 3. 将它保存到缓存
  // 4. 使用 module.load() 加载指定的模块文件，
  //    读取文件内容之后，使用 module.compile() 执行文件代码
  // 5. 如果加载/解析过程报错，就从缓存删除该模块
  // 6. 返回该模块的 module.exports
};
```

上面的第4步，采用`module.compile()`执行指定模块的脚本，逻辑如下:

```javascript
Module.prototype._compile = function(content, filename) {
  // 1. 生成一个require函数，指向module.require
  // 2. 加载其他辅助方法到require
  // 3. 将文件内容放到一个函数之中，该函数可调用 require
  // 4. 执行该函数
};
```

一旦require函数准备完毕，整个所要加载的脚本内容，就被放到一个新的函数之中，这样可以避免污染全局环境。该函数的参数包括require、module、exports，以及其他一些参数。

```javascript
(function (exports, require, module, __filename, __dirname) {
  // YOUR CODE INJECTED HERE!
});
```

Module._compile方法是同步执行的，所以Module._load要等它执行完成，才会向用户返回module.exports的值。

#### 模块缓存(Node)

第一次加载某个模块时，Node会缓存该模块。以后再加载该模块，就直接从缓存取出该模块的`module.exports`属性。

如果想要多次执行某个模块，可以让该模块输出一个函数，然后每次require这个模块的时候，重新执行一下输出的函数。

所有缓存的模块保存在`require.cache`之中，如果想删除模块的缓存，可以通过以下方式:

```javascript
// 删除指定模块的缓存
delete require.cache[moduleName];

// 删除所有模块的缓存
Object.keys(require.cache).forEach(function(key) {
  delete require.cache[key];
})
```

注意，缓存是根据`绝对路径`识别模块的，如果同样的模块名，但是保存在不同的路径，require命令还是会重新加载该模块。

CommonJS模块的特点:
- 所有代码都运行在模块作用域，不会污染全局作用域。
- 模块可以多次加载，但是只会在第一次加载时运行一次，然后运行结果就被`缓存`了，以后再加载，就直接读取缓存结果。要想让模块再次运行，必须清除缓存。
- 输出是值的`拷贝`(即require返回的值是被输出的值的拷贝，模块内部的变化也不会影响这个值)。
- 采用`同步阻塞`的方式加载模块, 加载完成后才会执行后面的操作。

```javascript
// 依赖就近
var a = require('./a') // 执行到此时，a.js 同步下载并执行
a.doSomething()

var b = require('./b')
b.doSomething()
```

### AMD

`AMD`是"Asynchronous Module Definition"的缩写，意思就是"异步模块定义"。它采用`异步`方式加载模块，可以`并行`加载多个模块, 模块的加载不影响它后面语句的运行。所有依赖这个模块的语句，都定义在一个回调函数中，等到加载完成之后，这个回调函数才会运行, 应用于`浏览器端`。`RequireJS`库实现了AMD规范。

#### 模块定义

AMD规范中, 模块采用`define()`函数定义在闭包中:

```javascript
define(id?: String, dependencies?: String[], factory: Function|Object);
```

- id: 是模块的名字，它是可选的参数
- dependencies: 指定了所要依赖的模块列表，它是一个数组，也是可选的参数，每个依赖的模块的输出将作为参数一次传入 factory 中。如果没有指定 dependencies，那么它的默认值是 ["require", "exports", "module"]。
  ```javascript
  define(function(require, exports, module) {}）
  ```
- factory: 包裹模块的具体实现，它是一个函数或者对象。如果是函数，那么它的`返回值`就是模块的输出接口或值。

例子:

```javascript
// 依赖前置, 提前加载, 提前执行(依赖模块)
define(['a', 'b'], function(a, b) {
  // 在这里 a.js, b.js 已经下载并且执行好了
  function foo() {
    a.doSomething()
    b.doSomething()
  }
  // 对外输出接口
　return {
　　foo : foo
　};
});
```

也可以在模块定义内部引用依赖:

```javascript
define(function(require) {
    var $ = require('jquery');
    $('body').text('hello world');
});
```

AMD规范允许输出的模块`兼容CommonJS`规范，这时`define`方法需要写成下面这样：

```javascript
define(function (require, exports, module){
  var someModule = require("someModule");
  var anotherModule = require("anotherModule");

  someModule.doTehAwesome();
  anotherModule.doMoarAwesome();

  exports.asplode = function (){
    someModule.doTehAwesome();
    anotherModule.doMoarAwesome();
  };
});
```

#### 模块加载

AMD采用`require()`函数加载模块。require()函数接受两个参数。第一个参数是一个`数组`，表示所依赖的模块；第二个参数是一个`回调函数`，当前面指定的模块都加载成功后，它将被调用。加载的模块会以参数形式传入该函数，从而在回调函数内部就可以使用这些模块。

```javascript
require(['moduleA', 'moduleB', 'moduleC'], function (moduleA, moduleB, moduleC){
　// some code here
});
```

### CMD

CMD是"Common Module Definition"的缩写, CMD规范和 AMD 很相似，也是应用于`浏览器端`的`异步`加载方案. CMD和AMD最大的不同在于对依赖模块的`执行时机`处理不同:
- `AMD`推崇`依赖前置`, 在定义模块的时候就要声明其依赖的模块，依赖模块加载完成后会立即执行该模块(`提前执行`), 所有模块都加载执行完后会进入require的回调函数, 执行主逻辑. 依赖模块的执行顺序和书写顺序不一定一致.
- `CMD`推崇`就近依赖`, 加载完某个依赖模块后并不立即执行, 所有依赖模块加载完成后进入主逻辑, 遇到require语句的时候才执行对应的模块(`延迟执行`). 依赖模块的执行顺序和书写顺序是一致的.

CMD 与 CommonJS 和 Node.js 的 Modules 规范保持了很大的兼容性。`SeaJS`实现了CMD规范。

#### 模块定义

CMD同样使用`define()`函数来定义模块:

```javascript
define(factory);
```

define 接受 factory 参数，factory 可以是一个`函数`，也可以是一个`对象`或`字符串`。

factory 为对象、字符串时，表示模块的接口就是该对象、字符串。比如可以如下定义一个 JSON 数据模块：

```javascript
define({ "foo": "bar" });
```

`factory` 为函数时，表示是模块的构造方法。执行该构造方法，可以得到模块向外提供的接口。factory 方法在执行时，默认会传入三个参数：`require`、`exports` 和 `module`：

- `require` 是一个方法，接受模块标识作为唯一参数，用来获取其他模块提供的接口：require(id)
- `exports` 是一个对象，用来向外提供模块接口
- `module` 是一个对象，上面存储了与当前模块相关联的一些属性和方法

```javascript
define(function(require, exports, module) {
  // 依赖就近，提前加载, 延迟执行(依赖模块)
  var a = require("a");
  a.doSomething();
  var b = require("b");
  b.doSomething(); 

  // 对外提供 foo 属性
  exports.foo = 'bar';

  // 对外提供 doSomething 方法
  exports.doSomething = function() {};
});
```

注意不能直接给`exports`赋值:

```javascript
define(function(require, exports) {
  // 错误用法！！!
  exports = {
    foo: 'bar',
    doSomething: function() {}
  };
});
```

因为`exports` 仅仅是 `module.exports` 的一个引用。在 factory 内部给 `exports` 重新赋值时，并不会改变 `module.exports` 的值。因此给 `exports` 赋值是无效的，不能用来更改模块接口。

正确的写法是给`module.exports`赋值:

```javascript
define(function(require, exports, module) {
  // 正确写法
  module.exports = {
    foo: 'bar',
    doSomething: function() {}
  };
});
```

对外提供模块接口时, 除了给 `exports` 对象增加成员，还可以使用 `return`:

```javascript
define(function(require) {
  // 通过 return 直接提供接口
  return {
    foo: 'bar',
    doSomething: function() {}
  };
});
```

define 也可以接受两个以上参数: `define(id?, deps?, factory)`。字符串 id 表示模块标识，数组 deps 是模块依赖。, 比如：

```javascript
define('hello', ['jquery'], function(require, exports, module) {
  // 模块代码
});
```
id 和 deps 参数可以省略。省略时，可以通过构建工具自动生成。

注意：带 id 和 deps 参数的 define 用法不属于 CMD 规范，而属于 [Modules/Transport](https://github.com/cmdjs/specification/blob/master/draft/transport.md) 规范。

#### 模块加载

```javascript
// seajs的实现
seajs.use(['myModule.js'], function(my){

});
```

### ES6 Module

ECMAScript6 标准增加了 JavaScript 语言层面的模块体系定义。ES6 模块的设计思想，是尽量的`静态`化，使得`编译时`就能确定模块的依赖关系，以及输入和输出的变量。CommonJS 和 AMD 模块，都只能在`运行时`确定这些东西。

ES6模块通过`export`命令显式指定输出的代码，再通过`import`命令输入:

```javascript
// ES6模块
import { stat, exists, readFile } from 'fs';
```

上面代码的实质是从fs模块加载 3 个方法，其他方法不加载。即在`import`时可以指定加载某个输出值，而不是加载整个模块, 这种加载称为“`编译时加载`”或者`静态加载`，即 ES6 可以在编译时就完成模块加载，效率要比 CommonJS 模块的加载方式高。当然，这也导致了没法引用 ES6 模块本身，因为它不是对象。

而CommonJS 模块就是对象；即在输入时是先加载整个模块，生成一个对象(即`module.exports`属性)，然后再从这个对象上面读取方法，这种加载称为“`运行时加载`”。

#### 模块定义

`export`命令用于规定模块的对外接口。一个模块就是一个独立的文件。该文件内部的所有变量，外部无法获取。如果你希望外部能够读取模块内部的某个变量，就必须使用export关键字输出该变量。

```javascript
export var firstName = 'Michael';

// 等价写法
var firstName = 'Michael';
export { firstName };
```

可以使用`as`关键字对输出的变量进行重命名:

```javascript
function v1() { ... }
function v2() { ... }

export {
  v1 as streamV1,
  v2 as streamV2,
  v2 as streamLatestVersion // 重命名后，v2可以用不同的名字输出两次
};
```

`export`命令规定的是对外的接口，必须与模块内部的变量建立一一对应关系。

```javascript
// 报错, 直接输出1, 没有提供对外接口
export 1;

// 报错, 通过变量的方式仍然直接输出1, 没有提供对外接口
var m = 1;
export m;
```

`export`语句输出的接口，与其对应的值是`动态绑定`关系，即通过该接口，可以取到模块内部`实时`的值。这一点与 CommonJS 规范完全不同。CommonJS 模块输出的是值的缓存(`浅拷贝`)，不存在动态更新。

> JS 引擎对脚本静态分析的时候，遇到模块加载命令import，就会生成一个只读引用。等到脚本真正执行时，再根据这个只读引用，到被加载的那个模块里面去取值。

```javascript
export var foo = 'bar';
setTimeout(() => foo = 'baz', 500); // 500ms后输出变量foo的值变为baz
```

`export`命令可以出现在模块的任何位置，只要处于`模块顶层`就可以。如果处于`块级作用域`内，就会报错(`import`命令亦同)。

#### 模块加载

使用`export`命令定义了模块的对外接口以后，其他 JS 文件就可以通过`import`命令加载这个模块。

```javascript
// main.js
import { firstName, lastName, year } from './profile.js';

function setName(element) {
  element.textContent = firstName + ' ' + lastName;
}
```

`import`命令输入的变量都是`只读`的，因为它的本质是输入接口。也就是说，不允许在加载模块的脚本里面，改写接口(如果输入的变量是对象, 则改写对象的属性是允许的)。

```javascript
import {a} from './xxx.js'

a = {}; // Syntax Error : 'a' is read-only;
```

`import`命令具有`提升`效果，会提升到整个模块的`头部`，首先执行。因为`import`命令是`编译阶段`执行的，在代码运行之前。

```javascript
// 不会报错，因为import的执行早于foo的调用
foo();

import { foo } from 'my_module';
```

`import`语句会执行所加载的模块, 多次重复执行同一句import语句，也只会执行一次:

```javascript
// 执行lodash模块，但是不输入任何值
import 'lodash';
import 'lodash'; // 不会执行
```

除了指定加载某个输出值，还可以使用整体加载，即用星号（`*`）指定一个对象，所有输出值都加载在这个对象上面。

```javascript
import * as circle from './circle';

console.log('圆面积：' + circle.area(4));
console.log('圆周长：' + circle.circumference(14));
```

`export default`命令用于指定模块的默认输出。显然，一个模块只能有一个默认输出，因此`export default`命令只能使用一次。所以，`import`命令后面才不用加大括号，因为只可能唯一对应`export default`命令。

```javascript
// 第一组
export default function crc32() { // 输出
  // ...
}
import crc32 from 'crc32'; // 输入

// 第二组
export function crc32() { // 输出
  // ...
};
import {crc32} from 'crc32'; // 输入
```

本质上，`export default`就是输出一个叫做`default`的变量或方法，然后系统允许你为它取任意名字。所以，下面的写法是有效的。

```javascript
// modules.js
function add(x, y) {
  return x * y;
}
export {add as default};
// 等同于
// export default add;

// app.js
import { default as foo } from 'modules';
// 等同于
// import foo from 'modules';
```

如果在一个模块之中，先输入后输出同一个模块，import语句可以与export语句写在一起。

```javascript
export { foo, bar } from 'my_module';

// 可以简单理解为
import { foo, bar } from 'my_module';
export { foo, bar };

// 整体输出, 注意 export * 命令会忽略 my_module 模块的 default 方法
export * from 'my_module';
```

需要注意的是，写成一行以后，foo和bar实际上并没有被导入当前模块，只是相当于对外转发了这两个接口，导致当前模块不能直接使用foo和bar。

##### 动态加载模块

ES2020提案引入 `import()` 函数, 支持`动态(异步)加载`模块. import函数的参数specifier，指定所要加载的模块的位置。import命令能够接受什么参数，import()函数就能接受什么参数，两者区别主要是后者为动态加载。

```javascript
import(specifier)
```

`import()`返回一个 `Promise` 对象。`import()`允许模块路径`动态`生成。

### 几种规范对比

|规范|依赖关系确定时机|加载|应用|实现|优点|缺点|
|--|--|--|--|--|--|--|
|CommonJS|运行时|同步阻塞, 依赖就近, 模块输出的是值的浅拷贝|服务端|1. 服务端的nodejs <br> 2. Browserify: 浏览器端的CommonJS实现|1. 服务器端模块便于重用 <br>2. 简单并容易使用|1. 同步的模块加载方式不适合在浏览器环境中，同步意味着阻塞加载，浏览器资源是异步加载的 <br>2. 不能非阻塞的并行加载多个模块
|AMD|运行时|异步加载, 依赖前置, 提前加载, 提前执行|浏览器端|1. RequireJS<br> 2. curl|1. 适合在浏览器环境中异步加载模块 <br>2. 可以并行加载多个模块|1. 提高了开发成本，代码的阅读和书写比较困难，模块定义方式的语义不顺畅 <br>2. 不符合通用的模块化思维方式，是一种妥协的实现|
|CMD|运行时|异步加载, 依赖就近, 提前加载, 延迟执行|浏览器端|1. Sea.js <br>2. coolie|1. 依赖就近，延迟执行 <br>2. 可以很容易在 Node.js 中运行| 1. 依赖 SPM 打包，模块的加载逻辑偏重|
|ES6 Module|编译时|编译时加载, 模块输出的是值的动态引用|服务端/浏览器端|1. Babel|1. 容易进行静态分析 <br> 2. 面向未来的 ECMAScript 标准|1. 原生浏览器端还没有实现该标准<br> 2. 全新的命令字，新版的 Node.js才支持|

## 工具

### webpack(v5.31.2)

webpack 打包应用程序时支持各种模块语法风格，包括 `ES6`, `CommonJS` 和 `AMD`。

#### ES6

webpack2 支持原生的ES6模块语法。webpack支持以下方法:

##### import

以静态的方式，导入另一个通过 export 导出的模块

##### export

默认导出整个模块，或具名导出模块

##### import()

动态地加载模块, 返回一个promise。调用 import() 之处，被作为分离的模块起点，意思是，被请求的模块和它引用的所有子模块，会分离到一个单独的 chunk 中。
```javascript
if ( module.hot ) {
  import('lodash').then(_ => {
    // Do something with lodash (a.k.a '_')...
  })
}
```

#### CommonJS

webpack支持以下CommonJS方法:

##### require

以`同步`的方式检索其他模块的导出。由编译器(compiler)来确保依赖项在最终输出 bundle 中可用。
```javascript
var $ = require("jquery");
var myModule = require("my-module");
```

##### require.resolve

以同步的方式获取模块的 ID。由编译器(compiler)来确保依赖项在最终输出 bundle 中可用。
```javascript
require.resolve(dependency: String)
```

#### AMD

webpack支持以下AMD方法:

##### define(通过 factory 方法导出)

```javascript
define([name: String], [dependencies: String[]], factoryMethod: function(...))
```

如果提供 dependencies 参数，将会调用 factoryMethod 方法，并（以相同的顺序）传入每个依赖项的导出。如果未提供 dependencies 参数，则调用 factoryMethod 方法时传入 require, exports 和 module（用于兼容）。如果此方法返回一个值，则返回值会作为此模块的导出。由编译器(compiler)来确保依赖项在最终输出 bundle 中可用。

```javascript
define(['jquery', 'my-module'], function($, myModule) {
  // 使用 $ 和 myModule 做一些操作……

  // 导出一个函数
  return function doSomething() {
    // ...
  };
});
```

##### define（通过 value 导出）

```javascript
define(value: !Function)
```

只会将提供的 value 导出。这里的 value 可以是除函数外的任何值。

```javascript
define({
  answer: 42
});
```

##### require（AMD 版本）

```javascript
require(dependencies: String[], [callback: function(...)])
```

与 require.ensure 类似，给定 dependencies 参数，将其对应的文件拆分到一个单独的 bundle 中，此 bundle 会被异步加载。然后会调用 callback 回调函数，并传入 dependencies 数组中每一项的导出。

```javascript
require(['b'], function(b) {
  var c = require("c");
});
```

#### webpack构建解析

```javascript
// index.js
var a = require('./out')
console.log(a.count)

// out.js
module.exports = {
  count: 1
}
```

webpack构建结果:

```javascript
/******/ (function(modules) { // webpackBootstrap
/******/ 	// The module cache
/******/ 	var installedModules = {};
/******/
/******/ 	// The require function
/******/ 	function __webpack_require__(moduleId) {
/******/
/******/ 		// Check if module is in cache
/******/ 		if(installedModules[moduleId]) {
/******/ 			return installedModules[moduleId].exports;
/******/ 		}
/******/ 		// Create a new module (and put it into the cache)
/******/ 		var module = installedModules[moduleId] = {
/******/ 			i: moduleId,
/******/ 			l: false,
/******/ 			exports: {}
/******/ 		};
/******/
/******/ 		// Execute the module function
/******/ 		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
/******/
/******/ 		// Flag the module as loaded
/******/ 		module.l = true;
/******/
/******/ 		// Return the exports of the module
/******/ 		return module.exports;
/******/ 	}
/******/
/******/
/******/ 	// expose the modules object (__webpack_modules__)
/******/ 	__webpack_require__.m = modules;
/******/
/******/ 	// expose the module cache
/******/ 	__webpack_require__.c = installedModules;
/******/  
/******/ 	// define getter function for harmony exports
/******/ 	__webpack_require__.d = function(exports, name, getter) {
/******/ 		if(!__webpack_require__.o(exports, name)) {
/******/ 			Object.defineProperty(exports, name, {
/******/ 				configurable: false,
/******/ 				enumerable: true,
/******/ 				get: getter
/******/ 			});
/******/ 		}
/******/ 	};
/******/
/******/ 	// getDefaultExport function for compatibility with non-harmony modules
/******/ 	__webpack_require__.n = function(module) {
/******/ 		var getter = module && module.__esModule ?
/******/ 			function getDefault() { return module['default']; } :
/******/ 			function getModuleExports() { return module; };
/******/ 		__webpack_require__.d(getter, 'a', getter);
/******/ 		return getter;
/******/ 	};
/******/
/******/ 	// Object.prototype.hasOwnProperty.call
/******/ 	__webpack_require__.o = function(object, property) { return Object.prototype.hasOwnProperty.call(object, property); };
/******/
/******/ 	// __webpack_public_path__
/******/ 	__webpack_require__.p = "";
/******/
/******/ 	// Load entry module and return exports
/******/ 	return __webpack_require__(__webpack_require__.s = 0);
/******/ })
/************************************************************************/
/******/ ([
/* 0 */
/***/ (function(module, exports, __webpack_require__) {

"use strict";


var a = __webpack_require__(1);
console.log(a.count);

/***/ }),
/* 1 */
/***/ (function(module, exports, __webpack_require__) {

"use strict";


module.exports = {
  count: 1
};

/***/ })
/******/ ]);
//# sourceMappingURL=demo.js.map
```

参考资料:

[JavaScript 模块化七日谈](http://huangxuan.me/js-module-7day/#/)

[前端模块化开发那点历史--玉伯](https://github.com/seajs/seajs/issues/588)

[JaVaScript中的模块](https://www.yexiaochen.com/JaVaScript%E4%B8%AD%E7%9A%84%E6%A8%A1%E5%9D%97/)

[ECMAScript 6 入门--Module](https://es6.ruanyifeng.com/#docs/module)

[CommonJS规范--阮一峰](https://javascript.ruanyifeng.com/nodejs/module.html)

[Javascript模块化编程--阮一峰](http://www.ruanyifeng.com/blog/2012/10/javascript_module.html)

[Webpack 中文指南](https://zhaoda.net/webpack-handbook/index.html)

[CMD 模块定义规范](https://github.com/seajs/seajs/issues/242)

[前端模块化：CommonJS,AMD,CMD,ES6](https://juejin.cn/post/6844903576309858318)

[前端模块化（AMD、CommonJS、UMD）总结](https://zhuanlan.zhihu.com/p/75980415)

[JavaScript modules](https://v8.dev/features/modules)

[webpack解惑：require的五种用法](https://www.cnblogs.com/lvdabao/p/5953884.html)

[模块方法(module methods)--webpack](https://www.webpackjs.com/api/module-methods/)