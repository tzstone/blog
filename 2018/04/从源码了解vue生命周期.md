# 从源码了解vue生命周期

本文主要从源码了解`beforeCreate`, `created`, `beforeMount`, `mounted`四个生命周期阶段, 基于vue2.5.16.  
先贴一个自己画的图:  
<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/vue%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.jpg" width=500
 height=1000 align=center />

## `beforeCreate`和`created`

创建一个Vue实例首先会调用`this._init()`方法. 源码如下  
可以看到`beforeCreate`之前会进行一些状态的初始化, 在`beforeCreate`和`created`之间则进行对`provide`, `inject`, `props`, `methods`, `data`, `computed`, `watch`进行初始化.

<br />

引用官方的解释:

>`beforeCreate`: 在实例初始化之后, 数据观测 (data observer) 和 event/watcher 事件配置之前被调用。

> `created`: 在实例创建完成后被立即调用。在这一步，实例已完成以下的配置：数据观测 (data observer)，属性和方法的运算，watch/event 事件回调。然而，挂载阶段还没开始，$el 属性目前不可见。

```javascript
Vue.prototype._init = function(options) {
    ...
    initLifecycle(vm); // 一些生命周期状态的初始化
    initEvents(vm); // 事件监听对象和状态初始化
    initRender(vm); // createElement函数绑定到实例
    callHook(vm, 'beforeCreate');
    initInjections(vm); // provide 和 inject 主要为高阶插件/组件库提供用例, 这里不展开
    initState(vm); // 对props, methods, data, computed, watch进行初始化
    initProvide(vm); // resolve provide after data/props
    callHook(vm, 'created');
    ...
    
    vm.$mount(vm.$options.el);
}

// 一些生命周期状态的初始化
function initLifecycle (vm) {
  ...
  vm._watcher = null;
  vm._inactive = null;
  vm._directInactive = false;
  vm._isMounted = false;
  vm._isDestroyed = false;
  vm._isBeingDestroyed = false;
}

// 事件初始化
function initEvents (vm) {
  vm._events = Object.create(null);
  vm._hasHookEvent = false;
  ...
}

function initState (vm) {
  vm._watchers = [];
  var opts = vm.$options;
  if (opts.props) { initProps(vm, opts.props); } // 劫持props属性的set和get, 通过代理vm._props将props挂载到vm
  if (opts.methods) { initMethods(vm, opts.methods); } // 把methods的方法挂载带vue实例上, 通过this.methodname可以直接访问
  if (opts.data) {
    initData(vm); // 劫持data属性的set和get, 通过代理vm._data将data挂载到vm
  } else {
    observe(vm._data = {}, true /* asRootData */);
  }
  if (opts.computed) { initComputed(vm, opts.computed); } // 劫持computed属性的set和get, 订阅观察者
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch); // 订阅观察者
  }
}
```

<br />

## `beforeMount`和`mounted`

`this._init()`最后会调用`vm.$mount()`, 这里涉及到vue的渲染机制.  
`$mount()`先把`template/el`转换成`render`函数并赋值给`this.$options.render`, 接下来会调用`new Watcher()`及`updateComponent()`, 执行`vm._update(vm._render(), hydrating)`.  
 `vm._render()`会返回一个VNode, `vm._update(vnode)`对新旧vnode进行diff, 返回一个渲染后的真实的dom, 最后这个dom会赋值给`vm.$el`.

<br />

引用官方的解释:
>`beforeMount`: 在挂载开始之前被调用：相关的 render 函数首次被调用。

>`mounted`: el 被新创建的 vm.$el 替换，并挂载到实例上去之后调用该钩子。如果 root 实例挂载了一个文档内元素，当 mounted 被调用时 vm.$el 也在文档内。


```javascript
// 运行时构建的$mount函数
Vue.prototype.$mount = function (
  el,
  hydrating
) {
  el = el && inBrowser ? query(el) : undefined;
  return mountComponent(this, el, hydrating)
};

function mountComponent (
  vm,
  el,
  hydrating
) {
  vm.$el = el; // 设置$el, 此时el中的变量未被渲染
  ...
  callHook(vm, 'beforeMount');

  ...
  updateComponent = function () {
    vm._update(vm._render(), hydrating); // vm._render()调用vm.$options.render,返回VNode, vm._update(vnode)根据新旧vnode进行diff计算, 返回一个真实的dom并赋值给vm.$el, 到此挂载完成
  };

  ...
  new Watcher(vm, updateComponent, noop, null, true /* isRenderWatcher */);

  ...
  callHook(vm, 'mounted');
  ...
}

// 独立构建的$mount函数, 先保存之前的$mount方法, 然后进行重写
var mount = Vue.prototype.$mount;
Vue.prototype.$mount = function (
  el,
  hydrating
) {
  ...
  // resolve template/el and convert to render function

      var ref = compileToFunctions(template, {
        shouldDecodeNewlines: shouldDecodeNewlines,
        shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this);
      var render = ref.render;
      var staticRenderFns = ref.staticRenderFns;
      options.render = render;
      options.staticRenderFns = staticRenderFns;
  }
  return mount.call(this, el, hydrating)
};

var Watcher = function Watcher (
  vm,
  expOrFn,
  cb,
  options,
  isRenderWatcher
) {
	...
    this.getter = expOrFn;
	...

  // this.get里调用了this.getter方法, 即执行了expOrFn(上面的updateComponent)
  this.value = this.lazy
    ? undefined
    : this.get();
};
```

## 总结
* `beforecreate`: 此时`data`, `props`, `methods`等还没初始化, 无法调用, 也不知道能做啥.
* `created`: `data`, `props`, `methods`等初始化完成, 可以进行调用.(网上有些文章说`mounted`里才发请求, 其实`created`里面就可以了.)
* `beforeMount`: `vm.$el`已赋值, 但是el中的变量未渲染, 使用如`{{msg}}`之类的占位.
* `mounted`: `vm.$el`已被渲染完后的dom替换

<br />

贴一下官方生命周期的图:  
<img src="https://cn.vuejs.org/images/lifecycle.png" width=500
 height=1200 align=center />


参考资料:  
[实例生命周期钩子](https://cn.vuejs.org/v2/guide/instance.html#%E5%AE%9E%E4%BE%8B%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E9%92%A9%E5%AD%90)  
[【Vue源码探究二】从 $mount 讲起，一起探究Vue的渲染机制](https://segmentfault.com/a/1190000009467029)
