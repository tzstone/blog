# vuex

## 流程图

### vue 数据流

<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/vue-flow.png" width=450
 height=300 align=center />

### vuex 数据流: 全局单例模式(Vue 单例)

<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/vuex.png" width=700
 height=550 align=center />

## 使用方法

```javascript
// store/index.js
Vue.use(Vuex);
export default new Vuex.Store({
  state,
  actions,
  mutations,
  modules: {
    // ...
  },
  getters,
});

// main.js
import store from './store';
new Vue({
  el: '#app',
  router,
  store,
  components: { App },
  template: '<App/>',
});
```

## 看看源码做了什么

```javascript
// vuex.js v3.0.1
var Vue; // bind on install
function install(_Vue) {
  if (Vue && _Vue === Vue) {
    {
      console.error('[vuex] already installed. Vue.use(Vuex) should be called only once.');
    }
    return;
  }
  Vue = _Vue;
  applyMixin(Vue);
}

var applyMixin = function (Vue) {
  var version = Number(Vue.version.split('.')[0]);

  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit });
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    // ...
  }
};

// 从根组件向所有子组件注入$store对象
function vuexInit() {
  var options = this.$options;
  // store injection
  if (options.store) {
    this.$store = typeof options.store === 'function' ? options.store() : options.store;
  } else if (options.parent && options.parent.$store) {
    this.$store = options.parent.$store;
  }
}

var Store = function Store(options) {
  var this$1 = this;
  if (options === void 0) options = {};

  // Auto install if it is not done yet and `window` has `Vue`.
  // To allow users to avoid auto-installation in some cases,
  // this code should be placed here. See #731
  if (!Vue && typeof window !== 'undefined' && window.Vue) {
    install(window.Vue);
  }

  {
    assert(Vue, 'must call Vue.use(Vuex) before creating a store instance.');
    assert(typeof Promise !== 'undefined', 'vuex requires a Promise polyfill in this browser.');
    assert(this instanceof Store, 'Store must be called with the new operator.');
  }

  var plugins = options.plugins;
  if (plugins === void 0) plugins = [];
  var strict = options.strict;
  if (strict === void 0) strict = false;

  var state = options.state;
  if (state === void 0) state = {};
  if (typeof state === 'function') {
    state = state() || {};
  }

  // store internal state
  this._committing = false;
  // 全局action命名空间
  this._actions = Object.create(null);
  this._actionSubscribers = [];
  // 全局mutation命名空间
  this._mutations = Object.create(null);
  // 全局getter命名空间
  this._wrappedGetters = Object.create(null);
  // 模块树
  this._modules = new ModuleCollection(options);
  this._modulesNamespaceMap = Object.create(null);
  this._subscribers = [];
  this._watcherVM = new Vue();

  // bind commit and dispatch to self
  var store = this;
  var ref = this;
  var dispatch = ref.dispatch;
  var commit = ref.commit;
  this.dispatch = function boundDispatch(type, payload) {
    return dispatch.call(store, type, payload);
  };
  this.commit = function boundCommit(type, payload, options) {
    return commit.call(store, type, payload, options);
  };

  // strict mode
  this.strict = strict;

  // init root module.
  // this also recursively registers all sub-modules
  // and collects all module getters inside this._wrappedGetters
  installModule(this, state, [], this._modules.root);

  // initialize the store vm, which is responsible for the reactivity
  // (also registers _wrappedGetters as computed properties)
  resetStoreVM(this, state);

  // apply plugins
  plugins.forEach(function (plugin) {
    return plugin(this$1);
  });

  if (Vue.config.devtools) {
    devtoolPlugin(this);
  }
};

// 初始化module中的state, getters, mutations, actions, 默认注册到全局命名空间
function installModule(store, rootState, path, module, hot) {
  var isRoot = !path.length;
  var namespace = store._modules.getNamespace(path);

  // register in namespace map
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module;
  }

  // set state
  if (!isRoot && !hot) {
    var parentState = getNestedState(rootState, path.slice(0, -1));
    var moduleName = path[path.length - 1];
    store._withCommit(function () {
      Vue.set(parentState, moduleName, module.state);
    });
  }

  var local = (module.context = makeLocalContext(store, namespace, path));

  module.forEachMutation(function (mutation, key) {
    var namespacedType = namespace + key;
    registerMutation(store, namespacedType, mutation, local);
  });

  module.forEachAction(function (action, key) {
    var type = action.root ? key : namespace + key;
    var handler = action.handler || action;
    registerAction(store, type, handler, local);
  });

  // 将modules(包括子module)的getter注册到全局的_wrappedGetters
  module.forEachGetter(function (getter, key) {
    var namespacedType = namespace + key;
    registerGetter(store, namespacedType, getter, local);
  });

  module.forEachChild(function (child, key) {
    installModule(store, rootState, path.concat(key), child, hot);
  });
}

// 默认情况下(即模块不带命名空间), getNamespace返回空(rootModule的path为空数组), 模块内部的action, mutation, getter都是注册到全局命名空间, 对应_actions, _mutations, _wrappedGetters
ModuleCollection.prototype.getNamespace = function getNamespace(path) {
  var module = this.root;
  return path.reduce(function (namespace, key) {
    module = module.getChild(key);
    return namespace + (module.namespaced ? key + '/' : '');
  }, '');
};

Module.prototype.forEachGetter = function forEachGetter(fn) {
  if (this._rawModule.getters) {
    forEachValue(this._rawModule.getters, fn);
  }
};

Module.prototype.forEachChild = function forEachChild(fn) {
  forEachValue(this._children, fn);
};

function forEachValue(obj, fn) {
  Object.keys(obj).forEach(function (key) {
    return fn(obj[key], key);
  });
}

// type: namespace + key
// 将modules(包括子module)的getter注册到全局的_wrappedGetters
function registerGetter(store, type, rawGetter, local) {
  if (store._wrappedGetters[type]) {
    {
      console.error('[vuex] duplicate getter key: ' + type);
    }
    return;
  }
  store._wrappedGetters[type] = function wrappedGetter(store) {
    // 执行getter函数会访问state, 依赖收集
    return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters, // root getters
    );
  };
}

// 建立state与getters的联系
function resetStoreVM(store, state, hot) {
  var oldVm = store._vm;

  // bind store public getters
  store.getters = {};
  var wrappedGetters = store._wrappedGetters;
  var computed = {};
  // 将getter作为Vue实例的computed属性, 实现响应式
  forEachValue(wrappedGetters, function (fn, key) {
    // use computed to leverage its lazy-caching mechanism
    computed[key] = function () {
      return fn(store);
    };
    Object.defineProperty(store.getters, key, {
      get: function () {
        // 相当于访问computed[key]
        return store._vm[key];
      },
      enumerable: true, // for local getters
    });
  });

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  var silent = Vue.config.silent;
  Vue.config.silent = true;
  store._vm = new Vue({
    data: {
      $$state: state,
    },
    computed: computed,
  });
  Vue.config.silent = silent;

  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store);
  }

  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(function () {
        oldVm._data.$$state = null;
      });
    }
    Vue.nextTick(function () {
      return oldVm.$destroy();
    });
  }
}
```

参考资料:  
[Vuex 状态管理](https://ustbhuangyi.github.io/vue-analysis/v2/vuex/#%E4%BB%80%E4%B9%88%E6%98%AF-%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86%E6%A8%A1%E5%BC%8F-%EF%BC%9F)
