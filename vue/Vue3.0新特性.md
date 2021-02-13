# Vue3.0 新特性

## 重要优化

### Object.defineProperty -> Proxy

Object.defineProperty 是一个相对比较昂贵的操作，因为它直接操作对象的属性(对对象的属性做 get 和 set)，颗粒度比较小。将它替换为 es6 的 Proxy，在目标对象之上架了一层拦截，代理的是对象而不是对象的属性。这样可以将原本对对象属性的操作变为对整个对象的操作，颗粒度变大, 提升性能。

javascript 引擎在解析的时候希望对象的结构越稳定越好，如果对象一直在变，可优化性降低，proxy 不需要对原始对象做太多操作。

### Virtual DOM 重构

vdom 的本质是一个抽象层，用 javascript 描述界面渲染成什么样子。react 用 jsx，没办法检测出可以优化的动态代码，所以做时间分片，vue 中足够快的话可以不用时间分片。

传统 vdom 的性能瓶颈：

- 虽然 Vue 能够保证触发更新的组件最小化，但在单个组件内部依然需要遍历该组件的整个 vdom 树。
- 传统 vdom 的性能跟模版大小正相关，跟动态节点的数量无关。在一些组件整个模版内只有少量动态节点的情况下，这些遍历都是性能的浪费。
- JSX 和手写的 render function 是完全动态的，过度的灵活性导致运行时可以用于优化的信息不足

vue 的特点是底层为 Virtual DOM，上层包含有大量静态信息的模版。为了兼容手写 render function，最大化利用模版静态信息，vue3.0 采用了动静结合的解决方案，将 vdom 的操作颗粒度变小，`每次触发更新不再以组件为单位进行遍历, 而是只 diff 动态节点`，主要更改如下

- 将模版基于动态节点指令切割为嵌套的区块
- 每个区块内部的节点结构是固定的
- 每个区块只需要以一个 Array 追踪自身包含的动态节点

vue3.0 将 vdom 更新性能由 与模版整体大小相关-->与动态内容的数量相关。

### 事件缓存

知道在 vue2 中，针对节点绑定的事件，每次触发都要重新生成全新的 function 去更新。在 Vue3 中，提供了事件缓存对象 cacheHandlers，当 cacheHandlers 开启的时候，编译会自动生成一个内联函数，将其变成一个静态节点，这样当事件再次触发时，就无需重新创建函数直接调用缓存的事件回调方法即可。

开启 cacheHandlers：

```javascript
import { createVNode as _createVNode, toDisplayString as _toDisplayString, openBlock as _openBlock, createBlock as _createBlock } from 'vue';

const _hoisted_1 = { id: 'app' };

export function render(_ctx, _cache) {
  return (
    _openBlock(),
    _createBlock('div', _hoisted_1, [
      _createVNode(
        'button',
        {
          onClick: _cache[1] || (_cache[1] = ($event) => _ctx.confirmHandler($event)),
        },
        '确认',
      ),
      _createVNode('span', null, _toDisplayString(_ctx.vue3), 1 /* TEXT */),
    ])
  );
}
```

关闭 cacheHandlers：

```javascript
import { createVNode as _createVNode, toDisplayString as _toDisplayString, openBlock as _openBlock, createBlock as _createBlock } from 'vue';

const _hoisted_1 = { id: 'app' };

export function render(_ctx, _cache) {
  return _openBlock(), _createBlock('div', _hoisted_1, [_createVNode('button', { onClick: _ctx.confirmHandler }, '确认', 8 /* PROPS */, ['onClick']), _createVNode('span', null, _toDisplayString(_ctx.vue3), 1 /* TEXT */)]);
}
```

### 压缩包体积更小

当前最小化并被压缩的 Vue 运行时大小约为 20kB（2.6.10 版为 22.8kB）。Vue 3.0 捆绑包的大小大约会减少一半，即只有 10kB！

## 新功能/特性

### Function_based API

在 Vue 3 中，全局和内部 API 都经过了重构，并考虑到了 tree-shaking 的支持。因此，全局 API 现在只能作为 ES 模块构建的命名导出进行访问。例如:

```javascript
import { nextTick } from 'vue';

nextTick(() => {
  // 一些和DOM有关的东西
});
```

Function-based API 对比 Class-based API 有以下优点:

- 对 typescript 更加友好，typescript 对函数的参数和返回值都非常好，写 Function-based API 既是 javascript 又是 typescript，不需要任何的类型声明，typescript 可以自己做类型推导。

- 静态的 import 和 export 是 treeshaking 的前提，Function-based API 中的方法都是从全局的 vue 中 import 进来的。

- 函数内部的变量名和函数名都可以被压缩为单个字母，但是对象和类的属性和方法名默认不被压缩（为了防止引用出错）。

- 更灵活的逻辑复用。

目前如果我们要在组件之间共享一些代码，则有两个可用的选择：mixins 和作用域插槽（ scoped slots），但是它们都存在一些缺陷：

- mixins 的最大缺点在于我们对它实际上添加到组件中的行为一无所知。这不仅使代码变得难以理解，而且还可能导致名称与现有属性和函数发生冲突。

- 通过使用作用域插槽，我们确切地知道可以通过 v-slot 属性访问了哪些属性，因此代码更容易理解。这种方法的缺点是我们只能在模板中访问它，并且只能在组件作用域内使用。

高阶组件在 vue 中比较少，在 react 中引入是作为 mixins 的替代品，但是比 mixins 更糟糕，高阶组件可以将多个组件进行包装，子组件通过 props 接收数据，多个高阶组件一起使用，不知道数据来自哪个高阶组件，存在命名空间的冲突。而且高阶组件嵌套得越多，额外的组件实例就越多，造成性能损耗。

一个逻辑复用的例子:

![](https://pic3.zhimg.com/80/v2-226c4386879bd81a8f586c61c94ee012_1440w.jpg)

![](https://pic2.zhimg.com/80/v2-e5e8db999c583473e1ae8d0da62e03bd_1440w.jpg)

### Composition API

在 Vue2.x 中，组件的主要逻辑是通过一些配置项来编写，包括一些内置的生命周期方法或者组件方法，例如下面的代码：

```javascript
export default {
  name: 'test',
  components: {},
  props: {},
  data() {
    return {};
  },
  created() {},
  mounted() {},
  watch: {},
  methods: {},
};
```

这种基于配置的组件写法称为 Options API（配置式 API），Vue3 的一大核心特性是引入了 Composition API（组合式 API）,这使得组件的大部分内容都可以通过 setup()方法进行配置，同时 Composition API 在 Vue2.x 也可以使用，需要通过安装@vue/composition-api 来使用，代码如下：

```javascript
npm install @vue/composition-api
...
import VueCompositionApi from '@vue/composition-api';

Vue.use(VueCompositionApi);
```

Composition API 的目的: 同一个逻辑关注点相关的代码配置在一起(逻辑关注点分离)。

新的 setup 组件选项在创建组件之前执行，一旦 props 被解析，并充当合成 API 的入口点。

setup 选项是一个接受 props 和 context 的函数。此外，我们从 setup 返回的所有内容都将暴露给组件的其余部分 (计算属性、方法、生命周期钩子等等) 以及组件的模板。

### Teleport

Teleport 提供了一种干净的方法，允许我们控制在 DOM 中哪个父节点下渲染了 HTML，而不必求助于全局状态或将其拆分为两个组件。可用于处理模态框。

### <Suspense>

<Suspense>是一个特殊的组件，它将呈现回退内容，而不是对于的组件，直到满足条件为止，这种情况通常是组件 setup 功能中发生的异步操作或者是异步组件中使用。例如这里有一个场景，父组件展示的内容包含异步的子组件，异步的子组件需要一定的时间才可以加载并展示，这时就需要一个组件处理一些占位逻辑或者加载异常逻辑，要用到<Suspense>，例如：

```code
<Suspense>
  <template >
    <Suspended-component />
  </template>
  <template #fallback>
    Loading...
  </template>
</Suspense>
```

上面代码中，假设<Suspended-component>是一个异步组件，直到它完全加载并渲染前都会显示占位内容：Loading，这就是<Suspense>的简单用法，该特性和 Fragment 以及<teleport>一样，灵感来自 React。

### 支持多根节点的组件

```code
<!-- Layout.vue -->
<template>
  <header>...</header>
  <main v-bind="$attrs">...</main>
  <footer>...</footer>
</template>
```

参考资料:

[vue 进阶之路 —— vue3.0 新特性](https://zhuanlan.zhihu.com/p/92143274)

[最全的 Vue3.0 升级指南](https://zhuanlan.zhihu.com/p/191216161)

[Vue3 迁移指南](https://v3.cn.vuejs.org/guide/migration/introduction.html#%E6%A6%82%E8%A7%88)
