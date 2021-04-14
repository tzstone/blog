# JS 执行上下文

本文是基于 ES5 和 ES6 规范, 旧版 ES3 规范描述参见[JavaScript. The Core.](http://dmitrysoshnikov.com/ecmascript/javascript-the-core/)

执行上下文是 JavaScript 代码`求值(evaluated)`和`执行(executed)`的环境的抽象概念。任何代码在 JavaScript 中运行时，它都是在执行上下文中运行的。

## 相关概念

### 作用域

通常，`作用域`用于管理来自程序不同部分的变量的`可见性`和`可访问性`。

一些与作用域相关的封装抽象(如名称空间、模块等)被发明出来，以提供更好的系统模块化，并避免命名变量冲突。同样的，也有函数的局部变量和块的局部变量(块级作用域)。这样的技术有助于提高抽象度和封装内部数据, 这样我们就不用关心内部的实现细节和担心命名变量的冲突。

作用域的概念让我们能够在一个程序中使用相同名称的变量，但它们可能具有不同的含义和值。从这个角度看:

> `作用域`是一个将`变量与值`相关联的`封闭`的`上下文`。

我们也可以说，作用域是让一个变量(甚至表达式)有其意义的`逻辑边界`。例如，一个全局变量、一个局部变量等等，它们通常反映了一个变量在其生存期所处的逻辑边界。

块和函数的概念将我们引向作用域的一个主要特性--作用域是一种`嵌套`结构(包含或者被包含其他作用域)。

ES6 之前的版本(即 ES2015)不支持块级作用域。ES6 标准新加入了 `let` 关键字来创建`块级作用域`变量。在此之前，块级作用域可以通过立即调用的函数表达式(IIFE)来实现。

```javascript
var x = 10;

if (true) {
  (function (x) {
    console.log(x); // 20
  })(20);
}

console.log(x); // 10
```

作用域的另一个主要特性是用于`变量解析`。事实上，由于在一个项目中很可能有几个程序员使用相同的变量名(例如，i 代表循环计数器)，我们应该知道如何确定具有相同名称的标识符所对应的正确的值。有两种概念上的方法，对应两种类型的作用域: 静态和动态。让我们来澄清一下。

#### 静态(词法)作用域

在静态作用域中，标识符引用自离它最近的词法环境。在本例中，词法这个词与程序文本(program text)的一个属性有关。也就是说，变量在源文本中出现的地方(即代码中的确切位置) -- 运行时引用的该变量将在这个作用域内被解析。

“静态”一词与程序`解析(parsing)`阶段确定标识符作用域(scope)的能力有关。也就是说，如果我们(通过查看代码)可以在程序启动之前就说明变量将在哪个范围内被解析，这就是一个静态作用域。

让我们看看下面的例子:

```javascript
var x = 10;
var y = 20;

function foo() {
  console.log(x, y);
}

foo(); // 10, 20

function bar() {
  var y = 30;
  console.log(x, y); // 10, 30
  foo(); // 10, 20
}

bar();
```

在这个例子中，变量 x 在全局作用域中定义，这意味着在运行时它也在全局作用域中解析，即值为 10。

对于名称为 y 的变量，代码中有两处定义。但是正如我们所说的，包含变量定义的`最近`的词法作用域有`最高优先级`，应该被首先考虑。因此，在 bar 函数中，y 变量被解析为 30。bar 函数的局部变量 y 遮蔽了全局作用域的同名变量 y 。

然而, 在 foo 函数中, 同名变量 y 被解析为 20 -- 即使它是在 bar 函数中被调用的, 而 bar 函数包含了另一个 y。这说明`标识符`的`解析`的独立于`调用者(caller)环境`的(在这个例子中, bar 是 foo 的 caller, foo 是 callee)。同样，在 foo 函数定义时，与 y 名称最接近的词法上下文是全局上下文。

### 函数的创建和应用规则

函数是相对于给定环境创建的。`函数对象`是由`代码(函数体)`和一个指向创建函数的`环境`的`指针`组成。

```javascript
// global "x"
var x = 10;

// function "foo" is created relatively
// to the global environment

function foo(y) {
  var z = 30;
  console.log(x + y + z);
}

// 伪代码
// create "foo" function
foo = functionObject {
  code: "console.log(x + y + z);"
  environment: {x: 10, outer: null}
};
```

这个函数对象如下图所示:

![A function](https://github.com/tzstone/MarkdownPhotos/raw/master/function-object.png)

注意，函数引用它的环境，而函数绑定的一个环境引用回函数对象。

当一个函数通过一组参数被调用(或应用)时, 将建立一个新的环境, 该环境绑定函数的形参到实参上, 并为局部变量创建绑定, 然后在该环境上下文中执行函数主体。新环境的`外围环境`就是函数对象指向的环境。

```javascript
// function "foo" is applied
// to the argument 20

foo(20);

// 伪代码
// create a new frame with formal
// parameters and local variables

fooFrame = {
  y: 20,
  z: 30,
  outer: foo.environment,
};

// and evaluate the code
// of the "foo" function

execute(foo.code, fooFrame); // 60
```

下图显示了使用 environment 的函数应用程序:

![Function application](https://github.com/tzstone/MarkdownPhotos/raw/master/function-application.png)

## 执行栈

`执行栈`，在其他编程语言中也称为`调用栈`，是一个具有 `LIFO(后进先出)`结构的栈，它用于存储在代码执行过程中创建的`所有执行上下文`。

当 JavaScript 引擎第一次遇到您的脚本时，它会创建一个`全局执行上下文`并将其推入`当前执行栈`。每当引擎发现一个`函数调用`时，它就为该函数创建一个`新的执行上下文`，并将其推到栈的`顶部`。

引擎执行其执行上下文位于栈顶部的函数。当这个函数完成时，它的执行栈从栈中`弹出`，`控制权`到达当前栈中它下面的上下文。

举个栗子:

```javascript
let a = 'Hello World!';
function first() {
  console.log('Inside first function');
  second();
  console.log('Again inside first function');
}
function second() {
  console.log('Inside second function');
}
first();
console.log('Inside Global Execution Context');
```

![execution_context_stack](https://github.com/tzstone/MarkdownPhotos/raw/master/execution_context_stack.png)

当上述代码在浏览器中加载时，Javascript 引擎会创建一个全局执行上下文，并将其推入当前执行栈。当遇到对 `first()`的调用时，Javascript 引擎会为该函数创建一个新的执行上下文，并将其推入当前执行栈的顶部。

当从 `first()`函数中调用 `second()`函数时，Javascript 引擎为该函数创建一个新的执行上下文，并将其推入当前执行栈的顶部。当 `second()`函数结束时，它的执行上下文从当前栈弹出，并且控制权到达它下面的执行上下文，这是 `first()`函数的执行上下文。

当 `first()`完成时，它的执行栈将从栈中移除，控制权将到达全局执行上下文。一旦所有代码执行完毕，JavaScript 引擎就会从当前栈中删除全局执行上下文。

## 执行上下文的类型

JavaScript 中有三种类型的执行上下文。

- `全局执行上下文(Global Execution Context)`

  这是默认或基本的执行上下文。`不在任何函数内部`的代码都在全局执行上下文中。它执行两件事:它创建一个`全局对象`，这是一个 `window 对象`(在浏览器的情况下)，并将 `this` 的值设置为等于这个全局对象。一个程序中只能有一个全局执行上下文。

- `函数执行上下文(Functional Execution Context)`

  每次`调用`函数时，都会为该函数创建一个全新的执行上下文。每个函数都有自己的执行上下文，但它是在函数被调用(invoked or called)时创建的。可以有任意数量的函数执行上下文。每当创建一个新的执行上下文时，它都会按照已定义的顺序执行一系列步骤，我将在本文后面讨论这些步骤。

- `Eval 函数执行上下文`

  `eval` 函数内部执行的代码也有它自己的执行上下文，但是由于 eval 通常不被 JavaScript 开发人员使用，因此在此不再赘述。

  在 ES5 中, 严格模式下的 eval 方法不再会在当前上下文创建变量。

## 执行上下文的结构

在 ES5 中, 执行上下文的组成组件有:

```javascript
ExecutionContextES5 = {
  ThisBinding: <this value>,
  VariableEnvironment: { ... },
  LexicalEnvironment: { ... },
}
```

我们看到一个上下文既有变量环境又有词法环境，这常常会引起规范读者的困惑。我们将简要地阐明它们，但这里只是简单地注意一下，这主要是为了区分`函数声明(FD)`和`函数表达式(FE)`的`[[Scope]]`属性的值。

注: 在抽象的定义和讨论中，并不会对 VariableEnvironment 和 LexicalEnvironment 进行严格的区分，因此我们通常只是说上下文的词法环境。

### this 绑定

this 值现在被称为 this 绑定。但是，除了术语的变化之外，语义上并没有太大的变化(除了在严格模式下，this 值可以是 undefined)。在全局上下文中，this 绑定仍然是全局对象本身。

在函数的执行上下文中，this 值仍然由函数被调用的方式决定。如果它是由引用([reference](http://dmitrysoshnikov.com/ecmascript/chapter-3-this/#-reference-type))调用的，那么该引用的基值(base value)将被用作 this 值，在所有其他情况下，`this` 的值将被设置为 `全局对象` 或者 `undefined`(在严格模式下)。

```javascript
var foo = {
  bar: function () {
    console.log(this);
  };
};

// --- Reference cases ---

// with a reference
foo.bar(); // `this` is foo - the base

var bar = foo.bar;

// with the reference
bar(); // `this` is the global or undefined, implicit base, 注意隐式的 base 下 this 可能为 global 或者 undefined(严格模式下), 显示的 base 下 this 则指向 base
this.bar(); // the same, explicit base, the global

// with also but another reference
bar.prototype.constructor(); // `this` is bar.prototype

// --- non-Reference cases ---

(foo.bar = foo.bar)(); // `this` is global or undefined
(foo.bar || foo.bar)(); // `this` is global or undefined
(function () { console.log(this); })(); // `this` is global or undefined
```

再次注意，在严格模式下，不可能再使用这种技巧获得全局对象:

```javascript
(function () {
  'use strict';
  var global = (function () {
    return this;
  })();
  console.log(global); // undefined!
})();
```

`箭头函数`的 `this` 值是特殊的: 它们的 this 是`词法(静态)`的，不是动态的。也就是说，它们的函数环境记录没有提供 this 值，而是从`父环境`中获取的(实际上, this 的指向与父环境被调用的方式有关)。

```javascript
var a = 'global';
var obj = {
  a: 'object',
  bar: function () {
    var foo = () => console.log(this.a);
    return foo;
  },
  foo: () => console.log(this.a),
};
obj.foo(); // global
obj.bar()(); // object, obj.bar()调用时创建的执行上下文中的this指向obj, 其词法环境作为obj.bar()()调用时创建的词法环境的父环境

var baz = { a: 'baz' };
baz.bar = obj.bar;
baz.bar()(); // baz

baz.bar = obj.bar(); // 理由同上
baz.bar(); // object
```

### 变量环境(VariableEnvironment)

[官方 ES6 文档](http://ecma-international.org/ecma-262/6.0/)将词法环境定义为:

> 词法环境是一种规范类型，用于根据 ECMAScript 代码的词法嵌套结构定义`标识符`与特定`变量`和`函数`的`关联`。词法环境由一个`环境记录`(Environment Record)和一个可能为空的`外部词法环境引用`(Reference to the outer environment)组成。

简而言之，`词法环境`是保存`标识符变量映射`的结构（此处的标识符是指变量/函数的名称，而变量是对实际对象（包括函数对象和数组对象）或原始值的引用）。

`变量环境组件`也是一个`词法环境`, 它是对上下文中的`变量`和`函数`的`初始存储`。其环境记录被用作数据存储，并在进入上下文阶段时填充。这正是 ES3 中的变量对象。

当进入一个函数的上下文时，会创建一个特殊的 `arguments` 对象来存储形参的值。在`严格模式`下，arguments 对象经历了几次变化，其中 arguments 不再与实参变量共享其属性值。属性 `callee`(对函数本身的引用)在 strict 模式下也已弃用。

对于以下代码:

```javascript
function foo(a) {
  var b = 20;
}

foo(10);
```

抽象地说，我们有以下 foo 函数上下文的变量环境组件:

```javascript
fooContext.VariableEnvironment = {
  environmentRecord: {
    arguments: { 0: 10, length: 1, callee: foo },
    a: 10,
    b: 20,
  },
  outer: globalEnvironment,
};
```

严格模式下, arguments 不与实参变量共享属性值:

```javascript
'use strict';

(function foo(x) {
  alert(arguments[0]); // 10
  arguments[0] = 20;

  alert(x); // 10, but not 20

  x = 30;
  alert(arguments[0]); // 20, but not 30
})(10);
```

那么什么是`词法环境`组件呢? 有趣的是，`最初`它只是`变量环境`组件的一个`拷贝`。

### 词法环境(LexicalEnvironment)

`VariableEnvironment` 和 `LexicalEnvironment` 本质上都是词法环境(先不管它们命名上造成的困惑)，也就是说，它们都是在`静态(词法上)`捕获了上下文中创建的内部函数的`外部绑定`。

正如我们刚才提到的，在初始化阶段(当`上下文被激活`时) `LexicalEnvironment` 组件只是 `VariableEnvironment` 的一个`副本`。考虑上面的例子，我们有:

```javascript
fooContext.LexicalEnvironment = copy(fooContext.VariableEnvironment);
```

然而，接下来在代码执行阶段发生的事情，与通过 `with` 语句和 `catch` 子句扩充词法环境有关(尽管，正如我们所说的，在 ES5 中它已经改为替换当前上下文环境，而不是像在 ES3 中那样扩充当前上下文环境)。with 语句和 catch 子句在它们执行时会`替换`当前的执行上下文环境。这种情况与`函数表达式`有密切关系。

从讨论函数创建规则中，我们知道闭包保存了创建它的上下文的词法环境。如果在 `with` 语句(或 `catch` 子句)中创建`函数表达式(FE)`，它应该保存`当前(新创建的)词法环境`。

如果我们替换了 VariableEnvironment 本身(而不是通过复制得到 LexicalEnvironment)，那么我们应该在 with 完成后将它恢复回来。然而，这意味着函数表达式不能引用在 with 语句执行期间创建的绑定，但是函数表达式需要这些 with 内的绑定。此外，我们不能替换 VariableEnvironment 本身，因为函数声明也可以在 with 语句中被调用，但与函数表达式相反，`函数声明`应该继续使用`初始状态`的绑定值，而不是来自 `with` 对象(我们将在下面的示例中看到它)。

> 这就是为什么以`函数声明(FD)`形式形成的闭包保存 `VariableEnvironment` 组件作为它们的[[Scope]]属性，而在这种情况下，`函数表达式(FE)`保存的正是 `LexicalEnvironment` 组件。这是将这两个组件分开的主要(实际上也是唯一的)原因。

如果考虑到 with 语句将从 ES(可能在下个版本 ES)中完全消失，就像在 ES5 严格模式中那样，那 ES 规范在这方面就不会那么混乱了。

再次说明: `函数表达式`保存 `LexicalEnvironment`, 因为它需要 with 执行期间创建的`动态绑定`, 而`函数声明`保存 `VariableEnvironment`, 根据规范它不能在`块级作用域`中被创建，并且会进行声明提升。

```javascript
var a = 10;

// FD
function foo() {
  console.log(a);
}

with ({ a: 20 }) {
  // FE
  var bar = function () {
    console.log(a);
  };

  foo(); // 10!, from VariableEnvrionment
  bar(); // 20,  from LexicalEnvrionment
}

foo(); // 10
bar(); // still 20
```

它的过程是这样的:

```javascript
// "foo" is created
foo[[Scope]] = globalContext[[VariableEnvironment]];

// "with" is executed
previousEnvironment = globalContext[[LexicalEnvironment]];

globalContext[[LexicalEnvironment]] = {
  environmentRecord: { a: 20 },
  outer: previousEnvironment,
};

// "bar" is created
bar[[Scope]] = globalContext[[LexicalEnvironment]];

// "with" is completed, restore the environment
globalContext[[LexicalEnvironment]] = previousEnvironment;
```

为了更清楚地了解这种区别，我们来写些不符合规范的代码，把函数声明放在块中。依照规范这是一个语法错误，但其实在今天所有的引擎实现都不会抛出错误，而是以自己的方式处理这种情况。Firefox 就有一套非标准的扩展叫做函数语句（FS）。多年来，在 ES5 的其他实现中，如 Chrome s V8，有以下行为:

```javascript
var a = 10;

with ({ a: 20 }) {
  // FD
  function foo() {
    // do not test in Firefox!
    console.log(a);
  }

  // FE
  var bar = function () {
    console.log(a);
  };

  foo(); // 10!, from VariableEnvrionment
  bar(); // 20,  from LexicalEnvrionment
}

foo(); // 10
bar(); // still 20
```

这里同样是上面提到的情况，函数声明(在本例中被提升到顶部, 声明提升)将 VariableEnvironment 组件保存为它的[[Scope]]，而函数表达式保存 with 执行时被替换后的 LexicalEnvironment 组件，即 with 创建的环境。

注意: 由于 ES6 规范了`块级函数声明`，在上面的例子中函数 foo 也捕获了创建时所在的词法环境(即 with 创建的词法环境), 输出 20。

在 `ES6` 中，`LexicalEnvironment` 和 `VariableEnvironment` 的一个区别在于前者用于存储`函数声明和变量(let 和 const)`绑定，而后者仅用于存储`变量(var)`绑定。

> 在抽象的定义和讨论中，不可能对这两个组件进行严格的区分(例如，当函数类型、声明或表达式的意义不是那么重要的环境时)，因此我们通常只是说上下文的词法环境。

然而，`LexicalEnvironment` 组件会参与`标识符解析`的过程。

一个词法环境可能包含多个内部词法环境。例如，如果一个函数包含两个嵌套函数，那么每个嵌套函数的词法环境中指向外部环境的引用都是它们的外层函数的环境。

```javascript
function foo() {
  var x = 10;
  function bar() {
    var y = 20;
    console.log(x + y); // 30
  }
  function baz() {
    var z = 30;
    console.log(x + y); // 40
  }
}

// ----- Environments -----
// "foo" environmnet
fooEnvironment = {
  environmentRecord: { x: 10 },
  outer: globalEnvironment,
};

// both "bar" and "baz" have the same outer
// environment -- the environment of "foo"
barEnvironment = {
  environmentRecord: { y: 20 },
  outer: fooEnvironment,
};

bazEnvironment = {
  environmentRecord: { z: 30 },
  outer: fooEnvironment,
};
```

#### 环境记录(Environment Record)

一个环境记录记录了在这个词法环境作用域(上下文)内创建的标识符绑定, 包括`变量`(局部变量和形参)和`函数声明`。有两种类型的环境记录: 声明式环境记录和对象环境记录。

##### 声明式环境记录(Declarative environment record)

声明式环境记录用于处理出现在`函数`范围(在这种情况下，这跟我们从 ES3 系列中了解到激活对象很像)和 `catch` 子句中的`变量`、`函数`、`形式参数`等。

注: catch 子句所对应的声明式环境记录中, 只有 `catch` 的`异常参数` e 会被记录到该`环境记录`中, 在 catch 子句中声明的变量则会进行提升, 不会记录到 catch 对应的环境记录中。

比如:

```javascript
// all: "a", "b" and "c"
// bindings are bindings of
// a declarative record
function foo(a) {
  var b = 10;
  function c() {}
}

// catch
try {
  ...
} catch (e) { // "e" is a binding of a declarative record
  ...
}
```

一般情况下，声明式记录的绑定被假定直接存储在实现的底层(例如，在虚拟机的寄存器中，从而提供快速访问)。这是与旧版 ES3 中使用的激活对象概念的主要区别。

也就是说，规范不要求(甚至间接不建议)将声明式记录实现为`简单对象`，在这种情况下，这是`低效`的。这一事实的结果是，声明式环境记录不会直接向用户级公开，这意味着我们不能将这些绑定作为记录的属性来访问。事实上，我们之前也不能，甚至在 ES3 中，激活对象也不能直接对用户访问(除了 Rhino 引擎实现中, 可以通过 \_\_parent\_\_ 属性暴露出来)。

声明式记录可能允许使用完整的`词法寻址技术`，即不考虑嵌套作用域的深度就可以直接访问所需的变量，而无需进行任何作用域链查找(如果存储是固定的且不可更改的，那么即使在编译时也可以知道所有的变量地址)。然而，ES5 规范并没有直接提到这一事实。

因此，我们需要理解的主要问题是，`为什么需要用声明式环境记录替换旧的激活对象概念，首先是实现的效率`。

###### eval 和 inner 函数可能会打破优化

eval 函数可能会打破这种优化，使我们退回到低效的处理，因为在这种情况下很难确定 eval 需要哪个绑定。

例如，V8 的实现(我猜还有其他的)优化了函数，既不创建 argument 对象(如果它不在函数体中)，也不捕获未使用的父变量。也就是说，这样的函数是轻量级的，并且只捕获使用过的词法变量。也就是说，如果没有使用任何父变量(即自由变量)，这些函数甚至不是闭包。

看一下不使用 eval 的函数的作用域变量部分(屏幕来自 Chrome 的 dev-tools):

![withoutEval](https://github.com/tzstone/MarkdownPhotos/raw/master/withoutEval.png)

然后是同一个函数，但里面调用了空的 eval:

![withEval](https://github.com/tzstone/MarkdownPhotos/raw/master/withEval.png)

因为我们事先不知道哪些数据将在 eval 中使用，所以我们应该创建所有你看到的重量级的东西，参数对象和闭包属性，也就是父环境。

此外，在后面的例子中，看看 outerFn。它还创建了参数对象，因为内部有一个函数，因此很难分析内部函数是否引用了父函数的参数。然而，这只是其中一个实现，尽管它允许我们看到哪些优化可以提供，以及如何取消这些优化。

##### 对象环境记录(Object environment record)

对象环境记录用于定义出现在`全局上下文`和 `with` 语句中的`变量`和`函数`的关联。这是一种上面提到过的通过一个简单对象来存储这些变量和函数标识符的低效率的方式。这种在一个上下文中用来存储绑定关系的对象称为`绑定对象(binding object)`。

对于`全局上下文`，`变量`与`全局对象`本身相`关联`。正因为如此，我们才能够将它们作为全局对象的属性来引用:

```javascript
var a = 10;
console.log(a); // 10

// "this" in the global context
// is the global object itself
console.log(this.a); // 10

// "window" is the reference to the
// global object in the browser environment
console.log(window.a); // 10
```

在 with 语句中，变量与 with 对象的属性相关联:

```javascript
with ({ a: 10 }) {
  console.log(a); // 10
}
```

每次执行 `with` 语句时，都会`创建`一个带有对象环境记录的`新词法环境`。因此，当前运行上下文的环境被设置为新环境的外部环境。然后将运行上下文的环境替换为这个新创建的环境。with 执行完成后，上下文环境恢复到以前的状态:

```javascript
var a = 10;
var b = 20;

with ({ a: 30 }) {
  console.log(a + b); // 50
}

console.log(a + b); // 30, restored
```

用伪代码形式表示:

```javascript
// initial state
context.lexicalEnvironment = {
  environmentRecord: { a: 10, b: 20 },
  outer: null,
};

// "with" executed
previousEnvironment = context.lexicalEnvironment;

withEnvironment = {
  environmentRecord: { a: 30 },
  outer: context.lexicalEnvironment,
};

// replace current environment
context.lexicalEnvironment = withEnvironment;

// "with" completed, restore the environment back
context.lexicalEnvironment = previousEnvironment;
```

`catch` 子句的效果是完全相同的，它也用`新创建的环境`替换了正在运行的上下文的`词法环境`。但与 with 语句相反，catch 子句创建的是声明式环境记录，而不是对象环境记录。

```javascript
var e = 10;

try {
  throw 20;
} catch (e) {
  // replace the environment
  console.log(e); // 20
}

// and now it's restored back
console.log(e); // 10
```

下面我们将看到这些 `with` 语句和 `catch` 子句创建的`临时环境`, 在其中使用函数表达式时发挥的作用(前面讨论的词法环境和变量环境的区别)。

由于对象环境记录的低效率，with 语句已经从 ES5 的严格模式中移除了。此外，with 语句经常会引起混淆(因为变量和函数声明的影响)，其中一些是真正令人困惑的情况。这也是为什么 with 语句从 ES5 的严格模式中删除的原因。

ES 的下个版本还计划将全局对象从作用域链的底部删除。也就是说，全局环境记录也是声明式的，而不是对象。对于计划的模块系统，全局绑定(如 parseInt、Math 等)只会被导入到全局上下文中，但从技术上讲，我们不能再将全局变量作为全局对象的属性引用，因为将不存在任何全局对象。

抽象地说，一个具有对象环境记录的环境，可以用这种方式来表示:

```javascript
environment = {
  // storage
  environmentRecord: {
    type: "object",
    bindingObject: {
      // storage
    }
  },
  // reference to the parent environment
  outer: <...>
};
```

规范中的`绑定对象`是`真实对象`(例如全局对象)的一种`反射`，但并不是原始对象的所有属性都包含在绑定对象中。例如，不是标识符的属性名不包含在绑定对象中，这很符合逻辑，因为我们不能在代码中将它们作为变量引用:

```javascript
// global properties
this['a'] = 10; // included in the binding object
this['hello world'] = 20; // isn't included

console.log(a); // 10, can refer
console.log(hello world); // cannot, syntax error
```

然而，规范中省略了绑定和原始对象同步的具体实现细节。

#### 外部环境引用

对外部环境的引用意味着它能够接触到外部的词法环境。这意味着，如果在当前词法环境中找不到变量，JavaScript 引擎可以在外部环境中寻找它们。全局环境是这条环境链的最后一环。

`标识符解析`的规则与原型链非常相似: `如果在自己的环境中找不到一个变量，就会尝试在父环境(外部环境)中查找它，在父环境的父环境中查找它，以此类推，直到找完整个环境链。`

看下面的例子:

```javascript
var x = 10;
function foo() {
  var y = 20;
}
```

现在我们有两个抽象环境，分别对应于全局上下文和 foo 函数的上下文:

```javascript
// environment of the global context
globalEnvironment = {
  environmentRecord: {
    // built-ins:
    Object: function,
    Array: function,
    // etc ...

    // our bindings:
    x: 10
  },
  outer: null // no parent environment
};

// environment of the "foo" function
fooEnvironment = {
  environmentRecord: {
    y: 20
  },
  outer: globalEnvironment
};
```

正如我们所看到的，外部引用用于将当前环境与父环境连接起来。当然，父环境可能有自己的外部链接。并且全局环境的外部链接被设置为 null。

抽象地说，词法环境在伪代码中是这样的:

```code
GlobalExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
    }
    outer: <null>,
    this: <global object>
  }
}
FunctionExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
    }
    outer: <Global or outer function environment reference>,
    this: <depends on how function is called>
  }
}
```

## 示例

让我们看一些例子来理解上面的概念:

```javascript
let a = 20;
const b = 30;
var c;
function multiply(e, f) {
  var g = 20;
  return e * f * g;
}
c = multiply(20, 30);
```

当执行上述代码时，JavaScript 引擎创建一个全局执行上下文来执行全局代码。因此，在创建阶段，全局执行上下文将看起来像这样:

```code
GlobalExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      a: < uninitialized >,
      b: < uninitialized >,
      multiply: < func >
    }
    outer: <null>,
    ThisBinding: <Global Object>
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      c: undefined,
    }
    outer: <null>,
    ThisBinding: <Global Object>
  }
}
```

### let、const 和 var

`let` 和 `const` 定义的变量在创建阶段没有任何与它们相关的值，但是 `var` 定义的变量被设置为 `undefined`。

这是因为，在创建阶段，会扫描代码中的变量和函数声明，而函数声明被完整地存储在环境中，变量最初被设置为 `undefined`(在使用 `var` 的情况下)或保持`未初始化`(在使用 `let` 和 `const` 的情况下)。这就是为什么你可以在声明变量之前访问 var 定义的变量(虽然未定义)，但在声明变量之前访问 let 和 const 变量会出现引用错误的原因。这就是我们所说的变量提升(hoisting)。

在执行阶段，如果 JavaScript 引擎无法在源代码中声明 let 变量的实际位置找到它的值，那么它将给它赋值 `undefined`。

在执行阶段，完成变量的赋值。因此，在执行阶段，全局执行上下文看起来是这样的。

```code
GlobalExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      a: 20,
      b: 30,
      multiply: < func >
    }
    outer: <null>,
    ThisBinding: <Global Object>
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      c: undefined,
    }
    outer: <null>,
    ThisBinding: <Global Object>
  }
}
```

当遇到对函数 multiply(20,30) 的调用时，将创建一个新的函数执行上下文来执行函数代码。因此，在创建阶段，函数执行上下文将看起来像这样:

```code
FunctionExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      Arguments: {0: 20, 1: 30, length: 2},
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>,
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      g: undefined
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>
  }
}
```

在此之后，执行上下文将经过执行阶段，这意味着对函数内部变量的赋值已经完成。所以函数的执行上下文在执行阶段看起来是这样的:

```code
FunctionExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      Arguments: {0: 20, 1: 30, length: 2},
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>,
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      g: 20
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>
  }
}
```

函数完成后，返回的值被存储在 c 中。因此全局词法环境被更新。在此之后，全局代码完成，程序结束。

参考资料:

[Understanding Execution Context and Execution Stack in Javascript](https://blog.bitsrc.io/understanding-execution-context-and-execution-stack-in-javascript-1c9ea8642dd0?gi=719892566e0)

[JavaScript. The Core: 2nd Edition](http://dmitrysoshnikov.com/ecmascript/javascript-the-core-2nd-edition/)

[ECMA-262-5 in detail. Chapter 3.1. Lexical environments: Common Theory.](http://dmitrysoshnikov.com/ecmascript/es5-chapter-3-1-lexical-environments-common-theory/)

[ECMA-262-5 in detail. Chapter 3.2. Lexical environments: ECMAScript implementation.](http://dmitrysoshnikov.com/ecmascript/es5-chapter-3-2-lexical-environments-ecmascript-implementation/)

[理解 JavaScript 中的执行上下文和执行栈](https://muyiy.cn/blog/1/1.1.html)

[JavaScript 深入之执行上下文](https://github.com/mqyqingfeng/Blog/issues/8)
