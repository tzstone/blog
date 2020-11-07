# vue 依赖收集与派发更新

## 流程梳理

### 初始化, 设置响应式对象

对 data, props 使用 defineReactive 进行数据劫持, 设置 get 和 set

对 组件渲染, watch, computed 使用 new Watcher 的方式进行依赖收集和更新派发

### 依赖收集

new Watch 的时候调用 this.get()方法, 将当前 watcher 赋值给全局的 Dep.target, 然后调用 watcher 的 getter 方法(由 new Watcher 时传入的 expOrFn 经过封装得来).

对于 watch 类型, expOrFn 为 watch 的 key, 调用 getter 即对 key 进行访问(如'a.b.c'); 对于渲染 watcher, expOrFn 为渲染函数; 对于 computed 类型, expOrFn 为 computed[key]对应的执行函数.

对于这三种类型, 执行 getter 方法都会触发对其所依赖的对象的访问(直接的对象访问或者执行函数中对对象的访问), 进入对象的 get 函数(在 defineReactive 中定义), 在 get 函数中获取到当前的 watcher 对象(Dep.target), 将其加入对象持有的依赖管理器 dep 中, 而后将 Dep.target 恢复到上一个状态, 完成依赖收集.

### 更新派发

当响应式对象被修改时, 触发对象的 set 方法(在 defineReactive 中定义), 调用 dep.notify()方法, 遍历执行 dep 收集的各个 watcher 的 update()方法, update()将当前 watcher 推入一个队列中, 在下一次 nextTick 中执行各个 watcher 的 run()方法(flushSchedulerQueue 方法).

run()方法会先通过 this.get()获取当前值, 并与旧值做对比, 满足条件时执行 watcher 的回调函数(cb). 只有 watch 类型的 watcher 才有 cb 回调, 其他两个类型的 cb 都为 noop.

对于 computed 类型 watcher 和对于渲染 watcher, 在 run()方法中执行 this.get()方法时, 会执行 this.getter()方法(同上述的依赖收集阶段). 对于 computed 类型 watcher, 会进行 computed[key]的重新计算; 对于渲染 watcher, 会执行 updateComponent 方法, 从而实现组件重新渲染.

## 源码解读

在 vue 进行初始化的时候, 会执行一个 initState 方法, 对 props, initMethods, data, computed, watch 等进行初始化操作

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

function initState(vm) {
  vm._watchers = [];
  var opts = vm.$options;
  if (opts.props) {
    // 劫持props属性的set和get, 通过代理vm._props将props挂载到vm
    initProps(vm, opts.props);
  }
  if (opts.methods) {
    // 把methods的方法挂载带vue实例上, 通过this.methodname可以直接访问
    initMethods(vm, opts.methods);
  }
  if (opts.data) {
    initData(vm); // 劫持data属性的set和get, 通过代理vm._data将data挂载到vm
  } else {
    observe((vm._data = {}), true /* asRootData */);
  }
  if (opts.computed) {
    // 劫持computed属性的set和get, 订阅观察者
    initComputed(vm, opts.computed);
  }
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch); // 订阅观察者
  }
}

function initComputed (vm, computed) {
  // $flow-disable-line
  var watchers = vm._computedWatchers = Object.create(null);
  // computed properties are just getters during SSR
  var isSSR = isServerRendering();

  for (var key in computed) {
    var userDef = computed[key];
    var getter = typeof userDef === 'function' ? userDef : userDef.get;

    if (!isSSR) {
      // create internal watcher for the computed property.
      // watcher的expOrFn为computed[key], 即computed[key]的执行函数
      // cb为noop
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      );
    }

    // ...
  }
}

function initWatch (vm, watch) {
  for (var key in watch) {
    var handler = watch[key];
    if (Array.isArray(handler)) {
      for (var i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i]);
      }
    } else {
      createWatcher(vm, key, handler);
    }
  }
}

function isPlainObject (obj) {
  return _toString.call(obj) === '[object Object]'
}

function createWatcher (vm, key, handler) {
  var options;
  if (isPlainObject(handler)) {
    options = handler;
    handler = handler.handler;
  }
  if (typeof handler === 'string') {
    handler = vm[handler];
  }
  vm.$watch(key, handler, options);
}

Vue.prototype.$watch = function (
  expOrFn,
  cb,
  options
) {
  var vm = this;
  options = options || {};
  options.user = true;
  // watch类型的Watcher的expOrFn为watch的key
  // cb为回调函数
  var watcher = new Watcher(vm, expOrFn, cb, options);
  if (options.immediate) {
    cb.call(vm, watcher.value);
  }
  return function unwatchFn () {
    watcher.teardown();
  }
}

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
  callHook(vm, 'beforeMount');
  // ...
  updateComponent = function () {
    vm._update(vm._render(), hydrating); // vm._render()调用vm.$options.render,返回VNode, vm._update(vnode)根据新旧vnode进行diff计算, 返回一个真实的dom并赋值给vm.$el, 到此挂载完成
  };
  // ...
  // expOrFn为渲染函数, cb为noop
  new Watcher(vm, updateComponent, noop, null, true /* isRenderWatcher */);
  // ...
  callHook(vm, 'mounted');
}

function initProps(vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {};
  const props = (vm._props = {});
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = (vm.$options._propKeys = []);
  const isRoot = !vm.$parent;
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false);
  }
  for (const key in propsOptions) {
    keys.push(key);
    const value = validateProp(key, propsOptions, propsData, vm);
    /* istanbul ignore else */
    // ...
    // 对props进行数据劫持
    defineReactive(props, key, value);
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      // 将对vm[key]的访问代理到vm._props[key]
      proxy(vm, `_props`, key);
    }
  }
  toggleObserving(true);
}

function initData(vm: Component) {
  let data = vm.$options.data;
  data = vm._data = typeof data === 'function' ? getData(data, vm) : data || {};
  if (!isPlainObject(data)) {
    data = {};
    process.env.NODE_ENV !== 'production' && warn('data functions should return an object:\n' + 'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function', vm);
  }
  // proxy data on instance
  const keys = Object.keys(data);
  const props = vm.$options.props;
  const methods = vm.$options.methods;
  let i = keys.length;
  while (i--) {
    const key = keys[i];
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(`Method "${key}" has already been defined as a data property.`, vm);
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(`The data property "${key}" is already declared as a prop. ` + `Use prop default value instead.`, vm);
    } else if (!isReserved(key)) {
      // 将对vm[key]的访问代理到vm._data[key]
      proxy(vm, `_data`, key);
    }
  }
  // observe data, 数据观测
  observe(data, true /* asRootData */);
}
```

```javascript
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop,
};
// 将target[key]的访问代理到将target[sourceKey][key]
export function proxy(target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter() {
    return this[sourceKey][key];
  };
  sharedPropertyDefinition.set = function proxySetter(val) {
    this[sourceKey][key] = val;
  };
  Object.defineProperty(target, key, sharedPropertyDefinition);
}
```

observe:

```javascript
/**
 * Attempt to create an observer instance for a value,
 * returns the new observer if successfully observed,
 * or the existing observer if the value already has one.
 */
export function observe(value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return;
  }
  let ob: Observer | void;
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__;
  } else if (shouldObserve && !isServerRendering() && (Array.isArray(value) || isPlainObject(value)) && Object.isExtensible(value) && !value._isVue) {
    ob = new Observer(value);
  }
  if (asRootData && ob) {
    ob.vmCount++;
  }
  return ob;
}

/**
 * Observer class that is attached to each observed
 * object. Once attached, the observer converts the target
 * object's property keys into getter/setters that
 * collect dependencies and dispatch updates.
 */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor(value: any) {
    this.value = value;
    // 依赖收集
    this.dep = new Dep();
    this.vmCount = 0;
    def(value, '__ob__', this);
    if (Array.isArray(value)) {
      const augment = hasProto ? protoAugment : copyAugment;
      augment(value, arrayMethods, arrayKeys);
      // 遍历数组调用observe
      this.observeArray(value);
    } else {
      // 遍历调用defineReactive, 设置对象getter和setter
      this.walk(value);
    }
  }

  /**
   * Walk through each property and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk(obj: Object) {
    const keys = Object.keys(obj);
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i]);
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray(items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i]);
    }
  }
}
```

defineReactive:

```javascript
// 定义响应式对象
export function defineReactive(obj: Object, key: string, val: any, customSetter?: ?Function, shallow?: boolean) {
  // 实例化一个Dep实例
  const dep = new Dep();

  const property = Object.getOwnPropertyDescriptor(obj, key);
  if (property && property.configurable === false) {
    return;
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get;
  const setter = property && property.set;
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key];
  }
  // 递归调用observe, 子属性也具备响应式
  let childOb = !shallow && observe(val);
  // 设置getter和setter
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      const value = getter ? getter.call(obj) : val;
      // 获取当前访问的watcher, 依赖收集, 订阅
      if (Dep.target) {
        // dep.depend()函数中调用Dep.target.addDep(this), 把当前dep放进watcher(Dep.target)的newDeps数组中, 最后调用dep.addSub(this), 又把watcher推进dep的subs数组中
        // 这样做的结果是dep.subs持有依赖该属性的watcher, 而watcher的newDeps(在后续cleanupDeps阶段会被赋值给deps)持有其所依赖的dep
        dep.depend();
        if (childOb) {
          childOb.dep.depend();
          if (Array.isArray(value)) {
            dependArray(value);
          }
        }
      }
      return value;
    },
    set: function reactiveSetter(newVal) {
      const value = getter ? getter.call(obj) : val;
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return;
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter();
      }
      if (setter) {
        setter.call(obj, newVal);
      } else {
        val = newVal;
      }
      // shallow为false时将newVal设置为响应式对象
      childOb = !shallow && observe(newVal);
      // 更新派发, 通知订阅者, 发布
      dep.notify();
    },
  });
}
```

Dep:

```javascript
// Dep定义: 对Watcher的管理, 包括订阅和发布
let uid = 0;

/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 */
export default class Dep {
  static target: ?Watcher; // 存放全局唯一Watcher
  id: number;
  subs: Array<Watcher>; // Watcher数组

  constructor() {
    this.id = uid++;
    this.subs = [];
  }

  addSub(sub: Watcher) {
    this.subs.push(sub);
  }

  removeSub(sub: Watcher) {
    remove(this.subs, sub);
  }

  depend() {
    if (Dep.target) {
      Dep.target.addDep(this);
    }
  }

  notify() {
    // stabilize the subscriber list first
    const subs = this.subs.slice();
    // 遍历执行watcher的update事件
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update();
    }
  }
}

// the current target watcher being evaluated.
// this is globally unique because there could be only one
// watcher being evaluated at any time.
Dep.target = null;
const targetStack = [];

export function pushTarget(_target: ?Watcher) {
  if (Dep.target) targetStack.push(Dep.target);
  Dep.target = _target;
}

export function popTarget() {
  Dep.target = targetStack.pop();
}
```

Watcher:

```javascript
// Watcher定义
let uid = 0;

/**
 * A watcher parses an expression, collects dependencies,
 * and fires callback when the expression value changes.
 * This is used for both the $watch() api and directives.
 */
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  computed: boolean;
  sync: boolean;
  dirty: boolean;
  active: boolean;
  dep: Dep;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  before: ?Function;
  getter: Function;
  value: any;

  constructor(vm: Component, expOrFn: string | Function, cb: Function, options?: ?Object, isRenderWatcher?: boolean) {
    this.vm = vm;
    if (isRenderWatcher) {
      vm._watcher = this;
    }
    vm._watchers.push(this);
    // options
    if (options) {
      this.deep = !!options.deep;
      this.user = !!options.user;
      this.computed = !!options.computed;
      this.sync = !!options.sync;
      this.before = options.before;
    } else {
      this.deep = this.user = this.computed = this.sync = false;
    }
    this.cb = cb;
    this.id = ++uid; // uid for batching
    this.active = true;
    this.dirty = this.computed; // for computed watchers
    this.deps = []; // watch持有的Dep实例数组
    this.newDeps = [];
    this.depIds = new Set();
    this.newDepIds = new Set();
    this.expression = process.env.NODE_ENV !== 'production' ? expOrFn.toString() : '';
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn;
    } else {
      this.getter = parsePath(expOrFn);
      if (!this.getter) {
        this.getter = function () {};
        process.env.NODE_ENV !== 'production' && warn(`Failed watching path: "${expOrFn}" ` + 'Watcher only accepts simple dot-delimited paths. ' + 'For full control, use a function instead.', vm);
      }
    }
    if (this.computed) {
      this.value = undefined;
      this.dep = new Dep();
    } else {
      // 实例化一个watch时, this.get()方法将当前watch赋值给Dep.target
      this.value = this.get();
    }
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get() {
    // 将当前watch赋值给Dep.target
    pushTarget(this);
    let value;
    const vm = this.vm;
    try {
      // this.getter为实例化watcher时传入的expOrFn, expOrFn可能是属性访问路径(比如watch里面的'a.b'属性嵌套类型访问), 也可能是函数, 比如渲染watcher的渲染函数和computed对象里属性对应的执行函数
      // 执行this.getter会触发对回调执行函数里依赖对象的访问, 触发依赖对象的get(在defineReactive里定义),
      // 此时当前watcher已被赋值给Dep.target, 所以在依赖对象的get中能获取当前watcher, 实现依赖收集
      value = this.getter.call(vm, vm);
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`);
      } else {
        throw e;
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        // 递归访问value, 触发所有子项的getter
        traverse(value);
      }
      // 当前watcher的依赖收集结束, 将Dep.target恢复成上一个状态
      popTarget();
      // 以新newDeps为准对deps进行订阅清理
      this.cleanupDeps();
    }
    return value;
  }

  /**
   * Add a dependency to this directive.
   */
  addDep(dep: Dep) {
    const id = dep.id;
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id);
      this.newDeps.push(dep);
      if (!this.depIds.has(id)) {
        dep.addSub(this);
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps() {
    let i = this.deps.length;
    // 以newDeps为准, 移除deps中的旧订阅
    while (i--) {
      const dep = this.deps[i];
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this);
      }
    }
    let tmp = this.depIds;
    this.depIds = this.newDepIds;
    this.newDepIds = tmp;
    this.newDepIds.clear();
    tmp = this.deps;
    this.deps = this.newDeps;
    this.newDeps = tmp;
    this.newDeps.length = 0;
  }

  update() {
    /* istanbul ignore else */
    if (this.computed) {
      // A computed property watcher has two modes: lazy and activated.
      // It initializes as lazy by default, and only becomes activated when
      // it is depended on by at least one subscriber, which is typically
      // another computed property or a component's render function.
      if (this.dep.subs.length === 0) {
        // In lazy mode, we don't want to perform computations until necessary,
        // so we simply mark the watcher as dirty. The actual computation is
        // performed just-in-time in this.evaluate() when the computed property
        // is accessed.
        this.dirty = true;
      } else {
        // In activated mode, we want to proactively perform the computation
        // but only notify our subscribers when the value has indeed changed.
        this.getAndInvoke(() => {
          this.dep.notify();
        });
      }
    } else if (this.sync) {
      this.run();
    } else {
      queueWatcher(this);
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run() {
    if (this.active) {
      this.getAndInvoke(this.cb);
    }
  }

  getAndInvoke(cb: Function) {
    const value = this.get();
    if (
      value !== this.value ||
      // Deep watchers and watchers on Object/Arrays should fire even
      // when the value is the same, because the value may
      // have mutated.
      isObject(value) ||
      this.deep
    ) {
      // set new value
      const oldValue = this.value;
      this.value = value;
      this.dirty = false;
      if (this.user) {
        try {
          cb.call(this.vm, value, oldValue);
        } catch (e) {
          handleError(e, this.vm, `callback for watcher "${this.expression}"`);
        }
      } else {
        cb.call(this.vm, value, oldValue);
      }
    }
  }
}

// queueWatcher
const queue: Array<Watcher> = [];
let has: { [key: number]: ?true } = {};
let waiting = false;
let flushing = false;
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher(watcher: Watcher) {
  const id = watcher.id;
  if (has[id] == null) {
    has[id] = true;
    // 非flushSchedulerQueue期间
    if (!flushing) {
      // 把watcher添加到队列中, 在nextTick中统一执行watcher回调
      queue.push(watcher);
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      // 从后往前找到合适的id插入位置, 相当于queue排序
      let i = queue.length - 1;
      while (i > index && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(i + 1, 0, watcher);
    }
    // queue the flush
    if (!waiting) {
      waiting = true;
      nextTick(flushSchedulerQueue);
    }
  }
}

// flushSchedulerQueue
let flushing = false;
let index = 0;
/**
 * Flush both queues and run the watchers.
 */
function flushSchedulerQueue() {
  flushing = true;
  let watcher, id;

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id);

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  // 每次都对queue.length进行求值, 因为watcher.run()可能会添加新的watcher
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index];
    if (watcher.before) {
      watcher.before();
    }
    id = watcher.id;
    has[id] = null;
    // 通过this.get()获取当前值, 满足条件下(新旧值不相等 等)执行watcher回调
    // new Watcher(vm, updateComponent, noop, null, true /* isRenderWatcher */);
    // 对应渲染watcher而言, 调用this.get()方法会执行getter方法(即updateComponent), 从而实现组件重新渲染
    watcher.run();
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1;
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn('You may have an infinite update loop ' + (watcher.user ? `in watcher with expression "${watcher.expression}"` : `in a component render function.`), watcher.vm);
        break;
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice();
  const updatedQueue = queue.slice();

  // 状态恢复
  resetSchedulerState();

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue);
  callUpdatedHooks(updatedQueue);

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush');
  }
}

// resetSchedulerState
const queue: Array<Watcher> = [];
let has: { [key: number]: ?true } = {};
let circular: { [key: number]: number } = {};
let waiting = false;
let flushing = false;
let index = 0;
/**
 * Reset the scheduler's state.
 */
function resetSchedulerState() {
  index = queue.length = activatedChildren.length = 0;
  has = {};
  if (process.env.NODE_ENV !== 'production') {
    circular = {};
  }
  waiting = flushing = false;
}

/**
 * Parse simple path.
 */
var bailRE = /[^\w.$]/;
function parsePath(path) {
  if (bailRE.test(path)) {
    return;
  }
  var segments = path.split('.');
  return function (obj) {
    for (var i = 0; i < segments.length; i++) {
      if (!obj) {
        return;
      }
      obj = obj[segments[i]];
    }
    return obj;
  };
}
```

参考资料:  
[深入响应式原理](https://ustbhuangyi.github.io/vue-analysis/v2/reactive/reactive-object.html#object-defineproperty)
