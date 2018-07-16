# vue 组件机制

页面代码如下:

```javascript
```

## 组件注册

全局的组件注册是使用了`Vue.component()`方法. 在引入`vue.js`文件时, 会通过`initGlobalAPI -> initAssetRegisters`全局注册`component, directive, filter`方法.

`Vue.component()`方法返回一个 Vue 的子类, 子类以`cid`属性进行区分. 同时该子类会缓存到`Vue.options.components`中.

`Vue.options`截图:
<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/vue.component.jpeg" align=center />

部分源码:

```javascript
// 全局注册vue.component, vue.directive, vue.filter三个方法
function initAssetRegisters(Vue) {
  /**
   * Create asset registration methods.
   */
  // ASSET_TYPES = ["component", "directive", "filter"]
  ASSET_TYPES.forEach(function(type) {
    Vue[type] = function(id, definition) {
      // 如果是创建component, 调用this.options._base.extend(即Vue.extend)方法
      // 返回一个Vue的子类
      if (type === "component" && isPlainObject(definition)) {
        definition.name = definition.name || id;
        definition = this.options._base.extend(definition);
      }
      // ...
      // 把extend的子类缓存到options.components中
      this.options[type + "s"][id] = definition;
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
  Vue.extend = function(extendOptions) {
    extendOptions = extendOptions || {};
    var Super = this;
    var SuperId = Super.cid;
    var cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {});
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId];
    }

    var name = extendOptions.name || Super.options.name;
    if ("development" !== "production" && name) {
      validateComponentName(name);
    }

    var Sub = function VueComponent(options) {
      this._init(options);
    };
    Sub.prototype = Object.create(Super.prototype);
    Sub.prototype.constructor = Sub;
    Sub.cid = cid++;
    Sub.options = mergeOptions(Super.options, extendOptions);
    Sub["super"] = Super;

    // For props and computed properties, we define the proxy getters on
    // the Vue instances at extension time, on the extended prototype. This
    // avoids Object.defineProperty calls for each instance created.
    if (Sub.options.props) {
      initProps$1(Sub);
    }
    if (Sub.options.computed) {
      initComputed$1(Sub);
    }

    // allow further extension/mixin/plugin usage
    Sub.extend = Super.extend;
    Sub.mixin = Super.mixin;
    Sub.use = Super.use;

    // create asset registers, so extended classes
    // can have their private assets too.
    ASSET_TYPES.forEach(function(type) {
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

function initMixin(Vue) {
  Vue.prototype._init = function(options) {
    var vm = this;
    // a uid
    vm._uid = uid$1++;

    var startTag, endTag;
    /* istanbul ignore if */
    if ("development" !== "production" && config.performance && mark) {
      startTag = "vue-perf-start:" + vm._uid;
      endTag = "vue-perf-end:" + vm._uid;
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
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      );
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
    callHook(vm, "beforeCreate");
    initInjections(vm); // resolve injections before data/props
    initState(vm);
    initProvide(vm); // resolve provide after data/props
    callHook(vm, "created");

    /* istanbul ignore if */
    if ("development" !== "production" && config.performance && mark) {
      vm._name = formatComponentName(vm, false);
      mark(endTag);
      measure("vue " + vm._name + " init", startTag, endTag);
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el);
    }
  };
}
```
