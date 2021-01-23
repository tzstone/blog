# bind

## 描述

bind() 函数会创建一个新的`绑定函数`（bound function，BF）。`绑定函数`是一个 exotic function object（`怪异函数对象`，ECMAScript 2015 中的术语），它`包装`了`原函数对象`。调用绑定函数通常会导致执行`包装函数`。绑定函数具有以下内部属性：

- `[[BoundTargetFunction]]` - 包装的函数对象
- `[[BoundThis]]` - 在调用包装函数时始终作为 this 值传递的值。
- `[[BoundArguments]]` - 列表，在对包装函数做任何调用都会`优先`用列表元素`填充`参数列表。
- `[[Call]]` - 执行与此对象关联的代码。通过函数调用表达式调用。内部方法的参数是一个 this 值和一个包含通过调用表达式传递给函数的参数的列表。

当调用绑定函数时，它调用 `[[BoundTargetFunction]]` 上的内部方法 `[[Call]]`，就像这样 `Call(boundThis, args)`。其中，`boundThis` 是 `[[BoundThis]]`，`args` 是 `[[BoundArguments]]` 加上通过函数调用传入的`参数列表`。

`绑定函数`也可以使用 `new 运算符`构造，它会表现为`目标函数`已经被`构建完毕`了似的。提供的 `this` 值会被`忽略`，但`前置参数`仍会提供给`模拟函数`。

## 语法

```js
function.bind(thisArg[, arg1[, arg2[, ...]]])
```

- thisArg

  调用绑定函数时作为 `this` 参数传递给`目标函数`的值。

  - 如果使用 `new 运算符`构造绑定函数，则`忽略`该值(提供的参数列表仍然会插入到构造函数调用时的参数列表之前)。
  - 当使用 bind 在 `setTimeout` 中创建一个函数（作为回调提供）时，作为 thisArg 传递的任何原始值都将转换为 `object`。
  - 如果 bind 函数的参数列表为空，或者 thisArg 是 null 或 undefined，`执行作用域`的 `this` 将被视为新函数的 `thisArg`。

- arg1, arg2, ...

  当目标函数被调用时，被预置入`绑定函数`的`参数列表`中的参数。

bind()方法`返回`一个`原函数`的`拷贝`，并拥有指定的 `this` 值和`初始参数`

## 应用示例

### 偏函数

bind() 的一个用法是使一个函数拥有`预设`的`初始参数`。只要将这些参数（如果有的话）作为 bind() 的参数写在 this 后面。当`绑定函数`被调用时，这些参数会被插入到`目标函数`的`参数列表`的`开始位置`，传递给绑定函数的参数会跟在它们后面。

```js
function addArguments(arg1, arg2) {
  return arg1 + arg2;
}
// 创建一个函数，它拥有预设的第一个参数
var addThirtySeven = addArguments.bind(null, 37);
var result2 = addThirtySeven(5);
// 37 + 5 = 42

var result3 = addThirtySeven(5, 10);
// 37 + 5 = 42 ，第二个参数被忽略
```

### 快捷调用

在你想要为一个需要特定的 this 值的函数创建一个捷径（shortcut）的时候，bind() 也很好用。

你可以用 Array.prototype.slice 来将一个类似于数组的对象（array-like object）转换成一个真正的数组，就拿它来举例子吧。你可以简单地这样写：

```js
var slice = Array.prototype.slice;
slice.apply(arguments);
```

用 bind()可以使这个过程变得简单。在下面这段代码里面，slice 是 Function.prototype 的 apply() 方法的绑定函数，并且将 Array.prototype 的 slice() 方法作为 this 的值。这意味着我们压根儿用不着上面那个 apply()调用了。

```js
// 与前一段代码的 "slice" 效果相同
var unboundSlice = Array.prototype.slice;
var slice = Function.prototype.apply.bind(unboundSlice);
slice(arguments);
```

### 配合 setTimeout

在默认情况下，使用 window.setTimeout() 时，this 关键字会指向 window （或 global）对象。当类的方法中需要 this 指向类的实例时，你可能需要显式地把 this 绑定到回调函数，就不会丢失该实例的引用。

## Polyfill

- 不支持使用 new 调用新创建的构造函数

```js
// Does not work with `new (funcA.bind(thisArg, args))`
if (!Function.prototype.bind)
  (function () {
    var slice = Array.prototype.slice;
    Function.prototype.bind = function () {
      var thatFunc = this, // 原函数
        thatArg = arguments[0]; // this绑定对象
      var args = slice.call(arguments, 1); // 预设参数列表
      if (typeof thatFunc !== 'function') {
        // closest thing possible to the ECMAScript 5
        // internal IsCallable function
        throw new TypeError('Function.prototype.bind - ' + 'what is trying to be bound is not callable');
      }
      // 返回新的绑定函数
      return function () {
        // 预设参数 + 调用入参
        var funcArgs = args.concat(slice.call(arguments));
        // 执行原函数, this指向绑定对象
        return thatFunc.apply(thatArg, funcArgs);
      };
    };
  })();
```

- 支持使用 new 调用新创建的构造函数

```js
//  Yes, it does work with `new (funcA.bind(thisArg, args))`
if (!Function.prototype.bind)
  (function () {
    var ArrayPrototypeSlice = Array.prototype.slice;
    Function.prototype.bind = function (otherThis) {
      if (typeof this !== 'function') {
        // closest thing possible to the ECMAScript 5
        // internal IsCallable function
        throw new TypeError('Function.prototype.bind - what is trying to be bound is not callable');
      }

      var baseArgs = ArrayPrototypeSlice.call(arguments, 1), // 预设参数列表
        baseArgsLength = baseArgs.length, // 预设参数长度
        fToBind = this, // 原函数
        fNOP = function () {},
        fBound = function () {
          baseArgs.length = baseArgsLength; // reset to default base arguments
          baseArgs.push.apply(baseArgs, arguments); // 预设参数 + 调用入参
          return fToBind.apply(fNOP.prototype.isPrototypeOf(this) ? this : otherThis, baseArgs);
        };

      // 构造函数调用
      if (this.prototype) {
        // Function.prototype doesn't have a prototype property
        fNOP.prototype = this.prototype;
      }
      fBound.prototype = new fNOP();

      return fBound;
    };
  })();
```

参考资料

[bind() -- MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)
