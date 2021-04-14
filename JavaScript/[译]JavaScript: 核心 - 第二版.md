# [译]JavaScript: 核心 - 第二版

> 原文地址：[JavaScript. The Core: 2nd Edition](http://dmitrysoshnikov.com/ecmascript/javascript-the-core-2nd-edition/)<br/>
> 作者: Dmitry Soshnikov

这是[JavaScript：核心]概述讲稿的第二版。核心概述讲座，专门介绍 ECMAScript 编程语言及其运行时系统的核心组件。

目标人群：有经验的程序员、专家。

本文的[第一版](http://dmitrysoshnikov.com/ecmascript/javascript-the-core/)涵盖了 JS 语言的通用方面，主要讲解了旧式 ES3 规范中的概念，并引用了一些 ES5 和 ES6(又名 ES2015)中的更新。

从 ES2015 开始，该规范改变了部分核心组件的描述和结构，引入了新的模型等。在这个版本中，我们会关注这些新的概念和术语，但是依然保留在规范各个版本中保持一致的最基本的 JS 结构。

本文涵盖了 ES2017+运行时系统。

> 注意: 最新版本的[ECMAScript 规范](https://tc39.github.io/ecma262/)可以在 TC-39 网站上找到。

我们从对象的概念开始讨论，这是 ECMAScript 的基础。

## 对象(Object)

ECMAScript 是一种基于原型的面向对象编程语言，其核心抽象是对象的概念。

> 定义 1. 对象: 对象是属性的集合，并且有一个单独的原型对象。原型可以是一个对象，也可以是 null 值。

让我们来看一个对象的简单例子。一个对象的原型由内部`[[prototype]]`属性引用，并通过 `__proto__` 属性暴露给用户级代码。

对于如下代码：

```javascript
let point = {
  x: 10,
  y: 20,
};
```

我们的结构具有两个显式的自有属性和一个隐式的 `__proto__` 属性，`__proto__` 属性是对 point 的原型的引用:

![Figure 1. A basic object with a prototype.](https://github.com/tzstone/MarkdownPhotos/raw/master/js-object.png)

> 注: 对象也可以存储 symbol。您可以在[本文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol)中获得更多有关 symbol 的信息。

原型对象通过`动态调度(dynamic dispatch)`机制实现继承。让我们考虑原型链的概念来详细了解这种机制。

## 原型(Prototype)

每个对象在创建时都会得到其原型（prototype）。如果原型没有显式设置，对象接收默认原型作为其继承对象。

> 定义 2. 原型: 原型是一个委托对象，用于实现基于原型的继承。

原型可以通过 `__proto__` 属性或 `Object.create` 方法显式设置:

```javascript
// Base object.
let point = {
  x: 10,
  y: 20,
};

// Inherit from `point` object.
let point3D = {
  z: 30,
  __proto__: point,
};

console.log(
  point3D.x, // 10, inherited
  point3D.y, // 20, inherited
  point3D.z, // 30, own
);
```

> 注: 默认情况下，对象接收 Object.prototype 作为其继承对象。

任何对象都可以作为另一个对象的原型，而原型本身也可以有自己的原型。如果一个原型有一个对其原型的非空引用，以此类推，这就被称为原型链。

> 定义 3. 原型链: 原型链是用于实现继承和共享属性的有限对象链。

![A prototype chain](https://github.com/tzstone/MarkdownPhotos/raw/master/prototype-chain.png)

规则非常简单: `如果一个属性在对象本身中找不到，就会尝试在原型中解析; 如果还找不到, 就到原型的原型中找, 以此类推, 直到找遍整个原型链。`

从技术上讲，这种机制称为`动态调度(dynamic dispatch)`或`委托(delegation)`。

> 定义 4. 委托: 用于解析继承链中的属性的机制。该过程在运行时发生，因此也称为动态调度。

> 注: 静态调度在编译时解析引用，动态调度在运行时解析引用。

如果一个属性最终在原型链中找不到，就返回 undefined:

```javascript
// An "empty" object.
let empty = {};

console.log(
  // function, from default prototype
  empty.toString,

  // undefined
  empty.x,
);
```

正如我们所看到的，默认对象实际上从来不是空的，它总是从 Object.prototype 继承一些东西。要创建一个没有原型的字典，我们必须显式地将其原型设置为 null:

```javascript
// Doesn't inherit from anything.
let dict = Object.create(null);

console.log(dict.toString); // undefined
```

动态调度机制允许继承链完全可变，提供了更改委托对象的能力:

```javascript
let protoA = { x: 10 };
let protoB = { x: 20 };

// Same as `let objectC = {__proto__: protoA};`:
let objectC = Object.create(protoA);
console.log(objectC.x); // 10

// Change the delegate:
Object.setPrototypeOf(objectC, protoB);
console.log(objectC.x); // 20
```

> 注: 尽管 `__proto__` 属性现在是标准化的，而且更容易用于解释，但在实践中更倾向于使用 API 方法进行原型操作，比如 Object.create, Object.getPrototypeOf,
> Object.setPrototypeOf，以及类似的 Reflect 模块。

在 Object.prototype 的示例中，我们看到同一个原型可以在多个对象之间共享。基于这个原则，在 ECMAScript 中实现了基于类的继承。让我们看看这个例子，看看 JS 中的"类"概念背后的机制。

## 类(Class)

当多个对象共享相同的初始状态和行为时，它们就形成了一种分类(classification)。

> 定义 5. 类: `类`是一个形式化的概念集合，它指定了`对象`的`初始状态`和`行为`。

如果我们需要从同一个原型继承多个对象，我们当然可以创建这个原型，并显式地从新创建的对象继承它:

```javascript
// Generic prototype for all letters.
let letter = {
  getNumber() {
    return this.number;
  },
};

let a = { number: 1, __proto__: letter };
let b = { number: 2, __proto__: letter };
// ...
let z = { number: 26, __proto__: letter };

console.log(
  a.getNumber(), // 1
  b.getNumber(), // 2
  z.getNumber(), // 26
);
```

我们可以在下面的图中看到这些关系:

![A shared prototype](https://github.com/tzstone/MarkdownPhotos/raw/master/shared-prototype.png)

然而，这显然很麻烦。而类正是为这个目的服务的--它是一个`语法糖`(即一个在语义上做同样事情的构造，但是以一个更好的语法形式)，它允许用方便的模式创建这样的多个对象:

```javascript
class Letter {
  constructor(number) {
    this.number = number;
  }

  getNumber() {
    return this.number;
  }
}

let a = new Letter(1);
let b = new Letter(2);
// ...
let z = new Letter(26);

console.log(
  a.getNumber(), // 1
  b.getNumber(), // 2
  z.getNumber(), // 26
);
```

> 注: ECMAScript 中基于类的继承是在基于原型的委托之上实现的。

> 注: 类只是一个理论上的概念。从技术上讲，它可以用 Java 或 C++中的静态调度来实现，也可以用 JavaScript、Python、Ruby 等中的动态调度(委派)来实现。

从技术上讲，一个`类`被表示为一对`构造函数` + `原型`。因此，构造函数创建对象，并自动为新创建的实例设置原型。这个原型存储在`<ConstructorFunction>.prototype` 属性中。

> 定义 6. 构造函数: 构造函数是用来创建实例并自动设置它们的原型的函数。

可以显式地使用构造函数。此外，在引入类的概念之前，JS 开发人员过去没有更好的选择(我们仍然可以在互联网上找到大量这样的遗留代码):

```javascript
function Letter(number) {
  this.number = number;
}

Letter.prototype.getNumber = function () {
  return this.number;
};

let a = new Letter(1);
let b = new Letter(2);
// ...
let z = new Letter(26);

console.log(
  a.getNumber(), // 1
  b.getNumber(), // 2
  z.getNumber(), // 26
);
```

虽然创建一个单级(single-level)构造函数非常容易，但是从父类继承的模式需要更多的`样板代码`。目前，这个样板代码作为实现细节被隐藏，这正是当我们在 JavaScript 中创建一个类时在底层发生的事情。

> 注: 构造函数只是基于类的继承的实现细节。

让我们看看对象和它们的类之间的关系:

![A constructor and objects relationship](https://github.com/tzstone/MarkdownPhotos/raw/master/js-constructor.png)

上图显示了每个对象都有一个相关联的原型。即使是构造函数(类)Letter 也有自己的原型，即 `Function.prototype`。注意, `Letter.prototype` 是 Letter 实例(即 a、b 和 z)的原型。

> 注: 任何对象的实际原型总是 `__proto__` 引用。构造函数的显式 prototype 属性只是对其实例的原型的引用; 在实例上，它仍然被 `__proto__` 引用。详情参见这里[ECMA-262-3 in detail. Chapter 7.2. OOP: ECMAScript implementation.](http://dmitrysoshnikov.com/ecmascript/chapter-7-2-oop-ecmascript-implementation/#explicit-codeprototypecode-and-implicit-codeprototypecode-properties)。

我们可以在[ECMA-262-3 in detail. Chapter 7.1. OOP: The general theory.](https://link.zhihu.com/?target=http%3A//dmitrysoshnikov.com/ecmascript/chapter-7-1-oop-general-theory/) 这篇文章中找到有关通用 OOP 概念的详细讨论（包括基于类、基于原型等的详细描述）。

现在我们已经理解了 ECMAScript 对象之间的基本关系，下面我们深入看看 JS 运行时系统。我们会看到，这里几乎所有东西也都可以被表示为对象。

## 执行上下文(Execution context)

为了执行 JS 代码并跟踪其运行时求值，ECMAScript 规范定义了执行上下文的概念。逻辑上的执行上下文使用一个堆栈来维护(即执行上下文堆栈)，栈与调用栈这个通用概念有关。

> 定义 7. 执行上下文: 执行上下文是用于跟踪代码运行时求值的规范设备。

ECMAScript 代码有几种类型: `全局代码`、`函数代码`、`eval 代码` 和 `模块(module)代码`; 每种代码都在其执行上下文中求值。不同的代码类型及其适当的对象可能会影响执行上下文的结构: 例如，生成器(generator)函数将其生成器对象保存在上下文中。

让我们考虑一个递归函数调用:

```javascript
function recursive(flag) {
  // Exit condition.
  if (flag === 2) {
    return;
  }

  // Call recursively.
  recursive(++flag);
}

// Go.
recursive(0);
```

当一个函数被`调用`时，一个新的执行上下文被`创建`，并被`压入堆栈`，此时它成为一个`活动`的执行上下文。当一个函数`返回`时，它的上下文从堆栈中`弹出`。

调用另一个上下文的上下文被称为`调用者(caller)`。相应地，被调用的上下文称为`被调用者(callee)`。在我们的例子中，递归函数在递归调用它自身时同时扮演两种角色: 被调用者和调用者。

> 定义 8. 执行上下文堆栈: 执行上下文堆栈是一种后进先出(LIFO)结构，用于维护控制流程和执行顺序。

对于上面的例子，我们有以下栈 "push-pop" 的修改:

![An execution context stack](https://github.com/tzstone/MarkdownPhotos/raw/master/execution-stack.png)

我们还可以看到，全局上下文(Global context)总是在堆栈的`底部`，它是在任何其他上下文执行之前创建的。

我们可以在对应的章([ECMA-262-3 in detail. Chapter 1. Execution Contexts.](http%3A//dmitrysoshnikov.com/ecmascript/chapter-1-execution-contexts/))找到执行上下文的更多详细内容。

一般来说，一个上下文的代码会一直运行到结束，然而，正如我们上面提到的，一些对象，如`生成器(generator)`，可能会违反堆栈的后进先出顺序。一个生成器函数可以`挂起`其正在执行的上下文，并在结束之前将其从堆栈中`移除`。一旦生成器再次被`激活`，它的上下文将被`恢复`，并再次被`压入堆栈`:

```javascript
function* gen() {
  yield 1;
  return 2;
}

let g = gen();

console.log(
  g.next().value, // 1
  g.next().value, // 2
);
```

这里的 yield 语句将值返回给调用者，并弹出上下文。在第二个 next 调用时，相同的上下文再次被压入堆栈，并继续执行。这样的上下文可能比创建它的调用者存在的时间长，因此违反了后进先出结构。

> 注：我们可以在这个文档([Iterators and generators](https%3A//developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators))中阅读有关 generator 和 iterator 的更多资料。

现在我们将讨论执行上下文的重要组成部分; 特别是，我们应该看到 ECMAScript 运行时如何管理由嵌套代码块创建的变量存储和作用域。这是词法环境的通常概念，它在 JS 中用于存储数据，并通过闭包机制解决 "Funarg (函数式参数)问题"。

## 环境(Environment)

每个`执行上下文`都有一个相关联的`词法环境`。

> 定义 9. 词法环境(lexical environment): `词法环境`是一种结构，用于定义出现在上下文中的`标识符与其值`之间的`关联`。每个环境都可以有一个对可选的`父环境的引用`。

因此，环境就是一个仓库(storage), 用于保存一个作用域中定义的变量、函数和类。

> 注: 您可以从[Essentials of Interpretation](https://www.youtube.com/playlist?list=PLGNbPb3dQJ_4WT_m3aI3T2LRf2R_FKM2k)的[appropriate lecture](https://www.youtube.com/watch?v=KRpYZBUkUsk)找到一个实现环境的例子。

从技术上讲，一个环境是由一个`环境记录(Environment Record)`(一个将标识符映射到值的实际存储表)和一个`对父对象的引用`(可以为 null)组成的。

对于以下代码:

```javascript
let x = 10;
let y = 20;

function foo(z) {
  let x = 100;
  return x + y + z;
}

foo(30); // 150
```

全局上下文和 foo 函数的上下文对应的环境结构如下所示:

![An environment chain](https://github.com/tzstone/MarkdownPhotos/raw/master/environment-chain.png)

从逻辑上讲，这让我们想起了我们前面讨论过的原型链。`标识符解析的规则非常相似: 如果在自己的环境中找不到一个变量，就会尝试在父环境中查找它，在父环境的父环境中查找它，以此类推，直到找完整个环境链`。

> 定义 10. 标识符解析(Identifier resolution): 在环境链中解析变量(绑定)的过程。未解析的绑定会导致 ReferenceError。

这解释了为什么将变量 x 解析为 100 而不是 10 -- 因为是直接在 foo 自己的环境中找到它；为什么我们可以访问参数 z -- 因为它也只存储在激活环境(activation environment)中；以及为什么我们可以访问变量 y -- 因为在父环境中可以找到它。

与原型类似，相同的父环境可以被几个子环境共享: 例如，两个全局函数共享相同的全局环境。

> 注：有关词法环境的详细信息可以参考[ECMA-262-5 in detail. Chapter 3.2. Lexical environments: ECMAScript implementation.](http://dmitrysoshnikov.com/ecmascript/es5-chapter-3-2-lexical-environments-ecmascript-implementation/).

环境记录根据类型而有所不同。有`对象(object)环境记录` 和 `声明式(declarative)环境记录`。在声明性式记录之上还有`函数环境记录` 和 `模块(module)环境记录`。每一种记录类型都有其特定的属性。然而，标识符解析的通用机制在所有环境中都是通用的，并且不依赖于记录的类型。

对象环境记录的一个例子是全局环境记录。这样的记录也有关联的`绑定对象(binding object)`，它可以存储来自该记录的一些属性，但不存储来自其它记录的属性，反之亦然。全局环境记录的绑定对象也可以被提供为 `this` 值。

```javascript
// Legacy variables using `var`.
var x = 10;

// Modern variables using `let`.
let y = 20;

// Both are added to the environment record:
console.log(
  x, // 10
  y, // 20
);

// But only `x` is added to the "binding object".
// The binding object of the global environment
// is the global object, and equals to `this`:

console.log(
  this.x, // 10
  this.y, // undefined!
);

// Binding object can store a name which is not
// added to the environment record, since it's
// not a valid identifier:

this['not valid ID'] = 30;

console.log(
  this['not valid ID'], // 30
);
```

如下图所示:

![A binding object](https://github.com/tzstone/MarkdownPhotos/raw/master/env-binding-object.png)

注意，`绑定对象`的存在是为了涵盖像 `var 声明`(全局环境下 var 声明的变量可以通过 `this` 访问) 和 `with 语句`等遗留结构，它们也将其对象提供为绑定对象。这是环境被表示为简单对象的历史原因。目前的环境模型已经优化了很多，但是我们不能再把绑定作为属性来访问了。

我们已经看到了环境是如何通过父链接关联起来的。现在我们来看看一个环境是如何比创建它的上下文存活得更久。这是我们将要讨论的闭包机制的基础。

## 闭包(Closure)

ECMAScript 中的函数是一等公民的。这个概念是函数式编程的基础，JavaScript 是支持函数式编程的。

> 定义 11. 一等函数（First-class Function）: 可以作为普通数据参与的函数: 可以存储在`变量`中，作为`参数`传递，或作为另一个函数的`返回值`返回。

与一等函数的概念有关的所谓 Funarg 问题(或函数参数的问题)。当一个函数必须处理自由变量时，问题就出现了。

> 定义 12. 自由变量（Free Variable）：函数使用的, 既不是函数的参数，也不是函数的局部变量的变量。

让我们来看看 Funarg 问题，看看在 ECMAScript 中是如何解决这个问题的。

考虑下面的代码片段:

```javascript
let x = 10;

function foo() {
  console.log(x);
}

function bar(funArg) {
  let x = 20;
  funArg(); // 10, not 20!
}

// Pass `foo` as an argument to `bar`.
bar(foo);
```

对于函数 foo，变量 x 就是自由变量。当 foo 函数被激活时(通过 funArg 参数)，它应该在哪里解析 x 绑定? 从创建函数的外部作用域，还是从调用函数的调用者的作用域? 正如我们所看到的，调用者 bar 函数也为 x 提供了值为 20 的绑定。

上面描述的用例被称为`向下 funarg 问题`，即确定正确解析绑定环境的模糊性: 它是创建时的环境，还是调用时的环境?

这可以通过使用`静态作用域(static scope)`(即创建时的作用域)来解决。

> 定义 13. 静态作用域: 一种语言实现了静态作用域，只要通过查看源代码就可以确定在哪个环境中解析绑定。

`静态作用域`有时也称为`词法作用域`，这也是`词法环境`这个名称的由来。

从技术上讲，`静态作用域`是通过捕获`函数创建`时所在的`环境`来实现的。(译注: 标识符的解析的与调用者(caller)环境无关)

> 注: 您可以在[本文](https://codeburst.io/js-scope-static-dynamic-and-runtime-augmented-5abfee6223fe)中阅读有关静态和动态作用域的内容。

在我们的示例中，foo 函数捕获的环境是全局环境(即父环境为全局环境)：

![A closure](https://github.com/tzstone/MarkdownPhotos/raw/master/closure.png)

我们可以看到，一个环境引用了一个函数，这个函数反过来又引用了该环境。

> 定义 14. 闭包: `闭包`是一个函数，它捕获它`被定义时`所处的`环境`。此外，此环境用于`标识符解析`。

> 注: 一个函数是在一个新的`激活环境(activation environment)`中被调用的, 这个环境存储了局部(local)变量和形参。该激活环境的父环境被设置为该函数的封闭环境(closured environment)，从而有了词法作用域的语义。

Funarg 问题的第二种子类型被称为`向上 funarg 问题`。这里唯一的区别是捕获的环境比创建它的上下文存活得更久。

让我们看看这个例子:

```javascript
function foo() {
  let x = 10;

  // Closure, capturing environment of `foo`.
  function bar() {
    return x;
  }

  // Upward funarg.
  return bar;
}

let x = 20;

// Call to `foo` returns `bar` closure.
let bar = foo();

bar(); // 10, not 20!
```

同样，从技术上讲，它与捕获定义环境的确切机制没有什么不同。在这种情况下，如果我们没有闭包，foo 的激活环境就会被销毁。但是我们捕获了它，所以它不能被释放，并且被保留以支持静态作用域语义。

通常情况下，开发者对闭包的理解是不完整的，他们通常只从向上 funarg 问题的角度来考虑闭包(实际上这样做更有意义)。然而，正如我们所看到的，向下和向上 funarg 问题的技术机制是完全相同的，就是`静态作用域`的机制。

正如我们上面提到的，与原型类似，相同的父环境可以在多个闭包之间共享。这允许访问和修改共享数据:

```javascript
function createCounter() {
  let count = 0;

  return {
    increment() {
      count++;
      return count;
    },
    decrement() {
      count--;
      return count;
    },
  };
}

let counter = createCounter();

console.log(
  counter.increment(), // 1
  counter.decrement(), // 0
  counter.increment(), // 1
);
```

因为两个闭包 increment 和 decrement 都是在包含 count 变量的作用域内创建的，所以它们共享这个父作用域。也就是说，捕获总是通过引用进行的，这意味着对整个父环境的引用被存储下来了。

我们可以在下面的图片中看到这一点:

![A shared environment](https://github.com/tzstone/MarkdownPhotos/raw/master/shared-environment.png)

有些语言可以按值捕获，复制捕获的变量，并且不允许在父作用域中更改它。然而，在 JS 中，要重复的是，它始终是对父作用域的引用。

> 注: 实现可能会优化这个步骤，不会捕获整个环境，只捕获使用过的自由变量，但在父作用域中仍然保持可变数据的不变。

有关闭包和 Funarg 问题的详细讨论，可以在对应的章节([ECMA-262-3 in detail. Chapter 6. Closures.](http://dmitrysoshnikov.com/ecmascript/chapter-6-closures/))中找到。

因此，所有`标识符`都是`静态作用域`的。然而，有一个值在 ECMAScript 中是`动态作用域`的。它是 `this` 值。

# This

this 值是一个特殊的对象，它`动态`地, `隐式`地传递给上下文的代码。我们可以把它看作一个隐式的`额外形参`，我们可以访问它，但不能改变它。

this 值的目的是为多个对象执行相同的代码。

> 定义 15. This: 一个隐式的上下文对象，可以从执行上下文的代码中访问，以便为多个对象应用相同的代码。

主要的用例是基于类的 OOP。一个实例方法(在原型上定义)存在于一个范例中，但是在这个类的所有实例中共享。

```javascript
class Point {
  constructor(x, y) {
    this._x = x;
    this._y = y;
  }
  getX() {
    return this._x;
  }
  getY() {
    return this._y;
  }
}

let p1 = new Point(1, 2);
let p2 = new Point(3, 4);

// Can access `getX`, and `getY` from
// both instances (they are passed as `this`).

console.log(
  p1.getX(), // 1
  p2.getX(), // 3
);
```

当 getX 方法被激活(activated)时，将创建一个新的环境来存储局部变量和形参。此外，函数环境记录获取传递的`[[ThisValue]]`，这是根据`调用函数的方式动态绑定`的。当它被 p1 调用时，this 值正好是 p1，而在第二种情况下，它是 p2。

this 的另一个应用是通用接口函数，它可以在 mixin 或 traits 中使用。

在下面的例子中，Movable 接口包含通用函数 move，它期望这个 mixin 的用户实现 \_x 和 \_y 属性:

```javascript
// Generic Movable interface (mixin).
let Movable = {
  /**
   * This function is generic, and works with any
   * object, which provides `_x`, and `_y` properties,
   * regardless of the class of this object.
   */
  move(x, y) {
    this._x = x;
    this._y = y;
  },
};

let p1 = new Point(1, 2);

// Make `p1` movable.
Object.assign(p1, Movable);

// Can access `move` method.
p1.move(100, 200);

console.log(p1.getX()); // 100
```

作为一种替代，mixin 也可以应用在原型级别，而不是像我们在上面的例子中所做的那样应用于每个实例。

为了展示 this 的动态性质，考虑一下这个例子，我们把它留给读者作为练习来解决:

```javascript
function foo() {
  return this;
}

let bar = {
  foo,
  baz() {
    return this;
  },
};

// `foo`
console.log(
  foo(), // global or undefined

  bar.foo(), // bar
  bar.foo(), // bar

  (bar.foo = bar.foo)(), // global
);

// `bar.baz`
console.log(bar.baz()); // bar

let savedBaz = bar.baz;
console.log(savedBaz()); // global
```

因为只通过查看 foo 函数的源代码，我们不能知道 this 在特定调用中会有什么值，所以我们说 this 值是动态作用域的。

> 注：我们可以在对应的章([ECMA-262-3 in detail. Chapter 3. This.](http://dmitrysoshnikov.com/ecmascript/chapter-3-this/))中，得到关于如何判断 this 值，以及为什么上面的代码会按那样的方式工作的详细解释。

`箭头函数`的 `this` 值是特殊的: 它们的 this 是`词法(静态)`的，不是动态的。也就是说，它们的函数环境记录没有提供 this 值，而是从`父环境`中获取的。

```javascript
var x = 10;

let foo = {
  x: 20,

  // Dynamic `this`.
  bar() {
    return this.x;
  },

  // Lexical `this`.
  baz: () => this.x,

  qux() {
    // Lexical this within the invocation.
    let arrow = () => this.x;

    return arrow();
  },
};

console.log(
  foo.bar(), // 20, from `foo`
  foo.baz(), // 10, from global
  foo.qux(), // 20, from `foo` and arrow
);
```

正如我们所说的，在全局上下文中，this 是全局对象(全局环境记录的绑定对象)。以前只有一个全局对象。在当前版本的规范中，可能有`多个全局对象`，它们是代码域(code realms)的一部分。我们来讨论一下这个结构。

## 域(Realm)

在求值之前，所有 ECMAScript 代码必须与一个域相关联。从技术上讲，`域`只是为`上下文`提供了一个`全局环境`。

> 定义 16. 域: `代码域`是一个封装了单独的`全局环境`的对象。

当创建执行上下文时，它与特定的代码域相关联，该代码域为该上下文提供了全局环境。这种关联保持不变。

> 注: 在浏览器环境中，域的一个直接对等物是 iframe 元素，它提供了一个自定义的全局环境。在 Node.js 中，它接近于 [vm 模块](https://nodejs.org/api/vm.html)的沙箱。

规范的当前版本没有提供显式创建域的能力，但是可以通过实现隐式地创建。不过已经有一个提案([tc39/proposal-realms](https://github.com/tc39/proposal-realms/)) 要暴露这个 API 给用户代码。

但从逻辑上讲，堆栈中的每个上下文总是与它的域相关联:

![A context and realm association](https://github.com/tzstone/MarkdownPhotos/raw/master/context-realm.png)

让我们看看使用 vm 模块的独立域示例:

```javascript
const vm = require('vm');

// First realm, and its global:
const realm1 = vm.createContext({ x: 10, console });

// Second realm, and its global:
const realm2 = vm.createContext({ x: 20, console });

// Code to execute:
const code = `console.log(x);`;

vm.runInContext(code, realm1); // 10
vm.runInContext(code, realm2); // 20
```

现在我们离 ECMAScript 运行时更近了一步。然而，我们仍然需要查看代码的入口点和初始化过程。这是由作业(jobs)和作业队列(job queues)机制管理的。

## Job

有些操作可以延迟，并在执行上下文堆栈上有可用点时执行。

> 定义 17. `Job`: Job 是一个抽象的操作，当没有其他的 ECMAScript 计算正在进行时，它会启动一个 ECMAScript 计算。

Jobs 在 job 队列(queues)中排队，在当前规范版本中有两种 job 队列: `ScriptJobs` 和 `PromiseJobs`。

ScriptJobs 队列上的初始 job 是程序的主入口点 - 加载和计算的初始脚本: 创建一个域，创建一个全局上下文并与这个域相关联，将其压入堆栈，并执行全局代码。

注意，ScriptJobs 队列同时管理脚本(scripts)和模块(modules)。

此外，该上下文还可以执行其他上下文，或对其他 job 进行排队。一个可以生成和排队 job 的例子是 promise。

当没有正在运行的执行上下文且执行上下文堆栈为空时，ECMAScript 实现会从 job 队列中移除第一个挂起的 job，创建一个执行上下文并开始执行。

> 注: job 队列通常由所谓的`事件循环(Event loop)`来处理。ECMAScript 标准没有指定事件循环，让它交给引擎实现，不过你可以在[这里](https://gist.github.com/DmitrySoshnikov/26e54990e7df8c3ae7e6e149c87883e4)找到一个演示示例。

示例:

```javascript
// Enqueue a new promise on the PromiseJobs queue.
new Promise((resolve) => setTimeout(() => resolve(10), 0)).then((value) => console.log(value));

// This log is executed earlier, since it's still a
// running context, and job cannot start executing first
console.log(20);

// Output: 20, 10
```

> 注: 您可以在[本文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)中阅读有关 promise 的更多信息。

async 函数可以等待 promise，因此它们也进入 promise job 队列:

```javascript
async function later() {
  return await Promise.resolve(10);
}

(async () => {
  let data = await later();
  console.log(data); // 10
})();

// Also happens earlier, since async execution
// is queued on the PromiseJobs queue.
console.log(20);

// Output: 20, 10
```

> 注：请在[这里](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)阅读更多有关 async 函数的知识。

现在我们非常接近当前 JS 领域的最终蓝图。现在我们将看到我们讨论过的所有组件的主要所有者: 代理(Agents)。

## 代理(Agent)

ECMAScript 采用代理模式实现并发和并行。代理模式非常接近 [Actor 模式](https://en.wikipedia.org/wiki/Actor_model) -- 一个具有消息传递风格的通信的轻量级进程。

> 定义 18. 代理: 代理是封装了执行上下文堆栈、job 队列集和代码域的一个抽象。

依赖代理的实现可以在同一个线程上运行，也可以在单独的线程上运行。浏览器环境中的 `Worker` 代理就是代理概念的一个例子。

代理之间的`状态`是相互`隔离`的，可以通过发送消息进行通信。一些数据可以在代理之间共享，例如 SharedArrayBuffers。代理也可以组合成代理集群(agent clusters)。

在下面的例子中，index.html 调用 agent-smith.js worker，传递共享的内存块:

```javascript
// In the `index.html`:

// Shared data between this agent, and another worker.
let sharedHeap = new SharedArrayBuffer(16);

// Our view of the data.
let heapArray = new Int32Array(sharedHeap);

// Create a new agent (worker).
let agentSmith = new Worker('agent-smith.js');

agentSmith.onmessage = (message) => {
  // Agent sends the index of the data it modified.
  let modifiedIndex = message.data;

  // Check the data is modified:
  console.log(heapArray[modifiedIndex]); // 100
};

// Send the shared data to the agent.
agentSmith.postMessage(sharedHeap);
```

如下是 worker 的代码：

```javascript
// agent-smith.js

/**
 * Receive shared array buffer in this worker.
 */
onmessage = (message) => {
  // Worker's view of the shared data.
  let heapArray = new Int32Array(message.data);

  let indexToModify = 1;
  heapArray[indexToModify] = 100;

  // Send the index as a message back.
  postMessage(indexToModify);
};
```

你可以在[这里](https://gist.github.com/DmitrySoshnikov/b75a2dbcdb60b18fd9f05b595135dc82)找到上面例子的完整代码。(注意，如果您在本地运行这个示例，那么在 Firefox 中运行它，因为 Chrome 出于安全原因不允许从本地文件加载 web worker)

下面是 ECMAScript 运行时的图片:

![ECMAScript runtime](https://github.com/tzstone/MarkdownPhotos/raw/master/agents-1.png)

这就是在 ECMAScript 引擎背后发生的事情。

现在我们讲完了。这就是我们可以在一篇概述文章中涵盖的关于 JS 核心的所有信息了。正如我们提到的，JS 代码可以被分组到模块中，对象的属性可以通过 Proxy 对象来跟踪，等等。在 JavaScript 语言的不同文档中，你可以找到许多用户级的细节。

这里我们尝试表示 ECMAScript 程序本身的逻辑结构，希望它能澄清这些细节。如果你有任何问题、建议或反馈，我会一如既往地在评论中讨论它们。

感谢 TC-39 的代表和规范的编辑帮助澄清本文。有关讨论可以在[这个推特跟帖](https://twitter.com/DmitrySoshnikov/status/930507793047592960)中找到。

祝学习 ECMAScript 顺利！
