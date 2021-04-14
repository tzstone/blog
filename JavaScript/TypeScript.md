# TypeScript

TypeScript 是 JavaScript 类型的超集，它可以编译成纯 JavaScript。换句话说，TypeScript 就是增强版的 JavaScript。这一点，从 TypeScript 的名字也可以看出，TypeScript = Type（类型） + JavaScript。

最初，JavaScript 是专为浏览器而设计的语言。起初只是用于处理基本的用户交互，比如表单处理、显示弹窗等。JS 本身作为一门动态语言，并没有类型系统，因为它压根就不是为了建造大型应用而设计的。

可是最近几年，JavaScript 飞速发展。Node.js 的出现，使得 JS 不再局限于浏览器中；而 SPA（单页应用）的流行，使得前端逐渐迈向工程化，项目的代码量急速膨胀。随之而来的问题就是，与 C#、Java 等更成熟的语言不同，JS 并不具备这样的规模化能力。松散的代码、纷乱的类型为项目带来了更多的不可控性和不确定性。

为了解决这个问题，大公司们开始寻求解决方案。而 TypeScript 就是 Microsoft 给出的回答，并且由 C# 之父 Anders Hejlsberg 领衔打造。

TypeScript 为 JavaScript 补全了构建大型应用时不可或缺的一块：类型系统，为开发者架上了一张安全网。

## 类型系统

我们都知道，JavaScript 中的数据可以分为基础数据类型和引用数据类型。基础数据类型包括：undefined、null、boolean、number、string、symbol（ES6 新增）。除了基础数据类型以外的，就是引用类型，包括：Object、Array、Date、Function、RegExp 等等。这里说的类型其实就是 TypeScript 中的 Type。只不过在 JavaScript 中，我们不需要把变量的类型明确写出来，而且，同一个变量可以赋值为不同类型的数据。但是在 TypeScript 中，每一个变量的类型都是确定的，不同类型的数据之间不能赋值。

```javascript
// JavaScript
let name = 'Hopsken';
name = 5; // 没啥问题

// TypeScript
let name: string = 'Hopsken';
name = 5; // Error: Type '5' is not assignable to type 'string'.
```

除此以外，TS 中，我们还可以指明函数的类型。通过声明函数应该接收什么类型的数据，返回什么类型的数据，可以有效地避免许多低级错误的出现。

> TypeScript 使用静态类型系统，只在编译时进行类型分析，最终编译出的 JS 源码中 **不含任何类型检查的代码**（自行添加的除外）。这一点要与运行时检查加以区分。

## 类型系统的益处

### 提前检测错误

静态类型系统的首要优点就是，能尽早发现逻辑错误，而不是到真正上线执行的时候才发现。JavaScript 松散的语法，在带来方便的同时，也让它变得很脆弱。通过上面的两个例子，可以感受到静态类型分析带来的优势。举个例子，相信不少人在 JavaScript 开发中，都遇到过『强制类型转换』的坑，而使用 TypeScript 则可以有效地避免这种问题，原因自然是不言而喻。

### 舒适的开发体验。

首先，类型系统的存在为很多辅助开发的工具提供了可能。比如说，当你调用函数时，现代的编辑器可以清楚地告诉你该函数需要几个参数、参数是什么类型的、哪些参数是可选的。这样可以省去大量查阅 API 的时间，提高开发效率。

其次，类型具有一定的自解释（self-explain）能力。而类型就像是对程序自身的注释。毕竟，代码写出来是让人读的。很多时候，光是看类型本身，我们就能理解某段程序的意图。与纯人工注释不同，随着项目的不断迭代，人工注释可能会越来越词不达意，但类型标注却可以始终忠实地反映程序本身的意义。
更强大的是，借助某些工具，可以根据类型标注自动生成文档。详情请参阅 [typedoc](https://typedoc.org/)。

### 更高的抽象性。

类型系统允许程序设计者对程序以较高层次的方式思考，将设计者从烦人的低层次实现中解脱出来，有一种提纲挈领的感觉。设计者可以通过设计子系统间的接口，来表达程序的逻辑。也就是说，让设计脱离实现，体现出一种 IDL（接口定义语言）的感觉，让程序设计回归本质。

参考资料:

[TypeScript 简明教程：认识 TypeScript](https://juejin.cn/post/6844903783869349902)

[TypeScript - 一种思维方式](https://zhuanlan.zhihu.com/p/63346965)

[TypeScript handbook — book](https://www.tslang.cn/docs/handbook/basic-types.html)
