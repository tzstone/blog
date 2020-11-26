# vue 组件机制

页面代码如下:

```html
<div id="app">
  <my-component> i am slot </my-component>
</div>
<script src="./vue.js"></script>
<script>
  var Child = {
    template: '<div>child content</div>',
  };
  Vue.component('my-component', {
    name: 'my-component',
    template: '<div><span>{{msg}}</span><Child></Child><slot></slot></div>',
    props: ['msg'],
    components: {
      Child,
    },
  });
  new Vue({
    el: '#app',
    data() {
      return {
        msg: 'i am a component',
      };
    },
  });
</script>
```

## 组件注册

全局的组件注册是使用了`Vue.component()`方法. 在引入`vue.js`文件时, 会通过`initGlobalAPI -> initAssetRegisters`全局注册`component, directive, filter`方法.

`Vue.component()`方法返回一个 Vue 的子类, 子类以`cid`属性进行区分. 同时该子类会缓存到`Vue.options.components`中.

`Vue.options`截图:
<img src="https://github.com/tzstone/MarkdownPhotos/raw/master/vue.component.jpeg" align=center />

```javascript
// 全局注册vue.component, vue.directive, vue.filter三个方法
function initAssetRegisters(Vue) {
  /**
   * Create asset registration methods.
   */
  // ASSET_TYPES = ["component", "directive", "filter"]
  ASSET_TYPES.forEach(function (type) {
    Vue[type] = function (id, definition) {
      // 如果是创建component, 调用this.options._base.extend(即Vue.extend)方法
      // 返回一个Vue的子类
      if (type === 'component' && isPlainObject(definition)) {
        definition.name = definition.name || id;
        definition = this.options._base.extend(definition);
      }
      // ...
      // 把extend的子类缓存到Vue.options.components中
      this.options[type + 's'][id] = definition;
      return definition;
    };
  });
}

function initExtend(Vue) {
  /**
   * Each instance constructor, including Vue, has a unique
   * cid. This enables us to create wrapped "child
   * constructors" for prototypal inheritance and cache them.
   */
  Vue.cid = 0;
  var cid = 1;

  /**
   * Class inheritance
   */
  Vue.extend = function (extendOptions) {
    extendOptions = extendOptions || {};
    var Super = this; // Vue
    var SuperId = Super.cid;
    var cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {});
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId];
    }

    var name = extendOptions.name || Super.options.name;
    if ('development' !== 'production' && name) {
      validateComponentName(name);
    }

    var Sub = function VueComponent(options) {
      // Vue.prototype._init, 对实例进行初始化
      this._init(options);
    };
    Sub.prototype = Object.create(Super.prototype);
    Sub.prototype.constructor = Sub;
    Sub.cid = cid++;
    Sub.options = mergeOptions(Super.options, extendOptions);
    Sub['super'] = Super;

    // For props and computed properties, we define the proxy getters on
    // the Vue instances at extension time, on the extended prototype. This
    // avoids Object.defineProperty calls for each instance created.
    // 将组件的props属性访问代理到Sub.prototype上
    if (Sub.options.props) {
      initProps$1(Sub);
    }
    // 代理拦截get/set, 依赖收集
    if (Sub.options.computed) {
      initComputed$1(Sub);
    }

    // allow further extension/mixin/plugin usage
    Sub.extend = Super.extend;
    Sub.mixin = Super.mixin;
    Sub.use = Super.use;

    // create asset registers, so extended classes
    // can have their private assets too.
    ASSET_TYPES.forEach(function (type) {
      Sub[type] = Super[type];
    });
    // enable recursive self-lookup
    if (name) {
      Sub.options.components[name] = Sub;
    }

    // keep a reference to the super options at extension time.
    // later at instantiation we can check if Super's options have
    // been updated.
    Sub.superOptions = Super.options;
    Sub.extendOptions = extendOptions;
    Sub.sealedOptions = extend({}, Sub.options);

    // cache constructor
    cachedCtors[SuperId] = Sub;
    return Sub;
  };
}

function initProps$1(Comp) {
  var props = Comp.options.props;
  for (var key in props) {
    proxy(Comp.prototype, '_props', key);
  }
}

function proxy(target, sourceKey, key) {
  sharedPropertyDefinition.get = function proxyGetter() {
    return this[sourceKey][key];
  };
  sharedPropertyDefinition.set = function proxySetter(val) {
    this[sourceKey][key] = val;
  };
  Object.defineProperty(target, key, sharedPropertyDefinition);
}

function initMixin(Vue) {
  Vue.prototype._init = function (options) {
    var vm = this;
    // a uid
    vm._uid = uid$1++;

    var startTag, endTag;
    /* istanbul ignore if */
    if ('development' !== 'production' && config.performance && mark) {
      startTag = 'vue-perf-start:' + vm._uid;
      endTag = 'vue-perf-end:' + vm._uid;
      mark(startTag);
    }

    // a flag to avoid this being observed
    vm._isVue = true;
    // merge options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options);
    } else {
      vm.$options = mergeOptions(resolveConstructorOptions(vm.constructor), options || {}, vm);
    }
    /* istanbul ignore else */
    {
      initProxy(vm);
    }
    // expose real self
    vm._self = vm;
    initLifecycle(vm);
    initEvents(vm);
    initRender(vm);
    callHook(vm, 'beforeCreate');
    initInjections(vm); // resolve injections before data/props
    initState(vm);
    initProvide(vm); // resolve provide after data/props
    callHook(vm, 'created');

    /* istanbul ignore if */
    if ('development' !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false);
      mark(endTag);
      measure('vue ' + vm._name + ' init', startTag, endTag);
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el);
    }
  };
}
```

## 渲染

由[vue 事件处理机制](https://github.com/tzstone/blog/blob/master/vue/vue%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6.md)我们知道, 模板会经过编译生成`ast`语法树, 再通过`generate(ast, options)`生成渲染的代码. 在本例中, 渲染`code`为:

code: `_c('div',{attrs:{"id":"app"}},[_c('my-component',[_v("\n i am slot\n ")])],1)`

`_c`就是`initRender`里的`vm._c`.

根据`code`进行渲染的具体过程参见[vue 事件处理机制](https://github.com/tzstone/blog/blob/master/vue/vue%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6.md)

```javascript
function initRender(vm) {
  vm._vnode = null; // the root of the child tree
  vm._staticTrees = null; // v-once cached trees
  var options = vm.$options;
  var parentVnode = (vm.$vnode = options._parentVnode); // the placeholder node in parent tree
  var renderContext = parentVnode && parentVnode.context;
  vm.$slots = resolveSlots(options._renderChildren, renderContext);
  vm.$scopedSlots = emptyObject;
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = function (a, b, c, d) {
    return createElement(vm, a, b, c, d, false);
  };
  // ...
}

// wrapper function for providing a more flexible interface
// without getting yelled at by flow
function createElement(context, tag, data, children, normalizationType, alwaysNormalize) {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children;
    children = data;
    data = undefined;
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE;
  }
  return _createElement(context, tag, data, children, normalizationType);
}

function _createElement(context, tag, data, children, normalizationType) {
  // ...
  var vnode, ns;
  if (typeof tag === 'string') {
    var Ctor;
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag);

    // 判断是否是html/svg保留标签
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      vnode = new VNode(config.parsePlatformTagName(tag), data, children, undefined, undefined, context);
    } else if (isDef((Ctor = resolveAsset(context.$options, 'components', tag)))) {
      // component
      // 如果是组件, 调用createComponent返回一个vnode
      vnode = createComponent(Ctor, data, context, children, tag);
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(tag, data, children, undefined, undefined, context);
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children);
  }
  // ...
}
```

## 事件

关于组件上的事件, 可查看[vue 事件处理机制](https://github.com/tzstone/blog/blob/master/vue/vue%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6.md)

## 参考资料

- [vue 源码解读－component 机制](https://segmentfault.com/a/1190000009721209)
