# vue 事件机制

在 vue 里面, 由于具体实现的不同, 可以将事件简单地分为两类, 普通 html 元素上的事件和组件上的事件. 但无论是哪种类型的事件, vue 做的事情都可以总结为:

- 编译模板, 识别模板绑定的事件
- 为 DOM 节点添加绑定事件
- 在恰当的时机触发这些事件

下面来看看 vue 是怎么做的:

## 普通 html 元素上的事件

监听普通 html 元素上的事件, 可以使用事件修饰符. 这里以使用`.stop`修饰符为例. 不使用修饰符的情况只在执行`genHandler`的时候有差异, 后面会讲到.

页面代码如下:

```html
<div id="app">
  <button @click.stop='handleClick'>click me</button>
</div>
<script src="./vue.js"></script>
<script>
  new Vue({
    el: '#app',
    methods: {
      handleClick() {
        console.log('click button')
      }
    }
  })
</script>
```

由[从源码了解 vue 生命周期](https://github.com/tzstone/blog/blob/master/vue/%E4%BB%8E%E6%BA%90%E7%A0%81%E4%BA%86%E8%A7%A3vue%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.md)我们知道, 在`created`之前, vue 会通过`initMethods(vm, opts.methods);`把 methods 的方法挂载到 vue 实例上, 而后会调用`vm.$mount(vm.$options.el);`开始编译模板乃至后面挂载`el`.

在上述例子中, 调用过程如下:

`$mount -> compileToFunctions -> compile -> baseCompile -> var ast = parse(template.trim(), options) -> var code = generate(ast, options) -> genElement -> genData$2 -> genHandler`

其中`ast`是将模板解析后得到的抽象语法树:
<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/vue%E4%BA%8B%E4%BB%B6%E6%9C%BA%E5%88%B6-ast.jpeg" align=center />
可以看到`ast`已经将`button`绑定的 click 事件解析到`events`对象中了. 由此, 对模板中的绑定事件进行识别已经完成了.

但由于 vue 提供了各种各样的事件/按键修饰符, 所以我们还必须对事件回调函数进行一些特殊处理.这里主要是调用了`genHandler`, 处理完之后, 最终得到的渲染代码`code`(`genElement`方法返回的结果)为:

`_c('div',{attrs:{"id":"app"}},[_c('button',{on:{"click":function($event){$event.stopPropagation();handleClick($event)}}},[_v("click me")])])`

部分源代码如下:

```javascript
var createCompiler = createCompilerCreator(function baseCompile(
  template,
  options
) {
  var ast = parse(template.trim(), options); // 解析模板, 得到抽象语法树ast
  if (options.optimize !== false) {
    optimize(ast, options);
  }
  var code = generate(ast, options); // 根据抽象语法树得到渲染函数render
  return {
    ast: ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  };
});

function generate(ast, options) {
  var state = new CodegenState(options);
  var code = ast ? genElement(ast, state) : '_c("div")'; // 得到上述的`code`
  return {
    render: "with(this){return " + code + "}", // 返回render函数
    staticRenderFns: state.staticRenderFns
  };
}

function genElement(el, state) {
  // ...
  // component or element
  var code;
  if (el.component) {
    code = genComponent(el.component, el, state);
  } else {
    var data = el.plain ? undefined : genData$2(el, state);

    var children = el.inlineTemplate ? null : genChildren(el, state, true);
    code =
      "_c('" +
      el.tag +
      "'" +
      (data ? "," + data : "") +
      (children ? "," + children : "") +
      ")";
  }
  // module transforms
  for (var i = 0; i < state.transforms.length; i++) {
    code = state.transforms[i](el, code);
  }
  return code;
}

function genData$2(el, state) {
  var data = "{";
  // ...
  // 本例中, 模板解析后的ast中button的events属性已经有值
  /*
    events: {
      click:{
        modifiers: {stop: true},
        value: 'handleClick'
      }
    }
  */
  // event handlers
  if (el.events) {
    data += genHandlers(el.events, false, state.warn) + ",";
  }
  if (el.nativeEvents) {
    data += genHandlers(el.nativeEvents, true, state.warn) + ",";
  }
  // ...
  // v-bind data wrap
  if (el.wrapData) {
    data = el.wrapData(data);
  }
  // v-on data wrap
  if (el.wrapListeners) {
    data = el.wrapListeners(data);
  }
  return data;
}

function genHandlers(events, isNative, warn) {
  var res = isNative ? "nativeOn:{" : "on:{";
  for (var name in events) {
    res += '"' + name + '":' + genHandler(name, events[name]) + ",";
  }
  return res.slice(0, -1) + "}";
}

function genHandler(name, handler) {
  if (!handler) {
    return "function(){}";
  }

  if (Array.isArray(handler)) {
    return (
      "[" +
      handler
        .map(function(handler) {
          return genHandler(name, handler);
        })
        .join(",") +
      "]"
    );
  }

  var isMethodPath = simplePathRE.test(handler.value);
  var isFunctionExpression = fnExpRE.test(handler.value);

  // 如果没有修饰符, 直接返回handler
  if (!handler.modifiers) {
    if (isMethodPath || isFunctionExpression) {
      return handler.value;
    }
    /* istanbul ignore if */
    return "function($event){" + handler.value + "}"; // inline statement
  } else {
    var code = "";
    var genModifierCode = "";
    var keys = [];
    for (var key in handler.modifiers) {
      // 处理修饰符, 将修饰符对应的代码片段添加到handler,
      // 如.stop就是将$event.stopPropagation();添加到handler
      if (modifierCode[key]) {
        genModifierCode += modifierCode[key];
        // left/right
        if (keyCodes[key]) {
          // 键盘处理
          keys.push(key);
        }
      } else if (key === "exact") {
        var modifiers = handler.modifiers;
        genModifierCode += genGuard(
          ["ctrl", "shift", "alt", "meta"]
            .filter(function(keyModifier) {
              return !modifiers[keyModifier];
            })
            .map(function(keyModifier) {
              return "$event." + keyModifier + "Key";
            })
            .join("||")
        );
      } else {
        keys.push(key);
      }
    }
    if (keys.length) {
      code += genKeyFilter(keys);
    }
    // Make sure modifiers like prevent and stop get executed after key filtering
    if (genModifierCode) {
      code += genModifierCode;
    }
    var handlerCode = isMethodPath
      ? handler.value + "($event)" // 传递$event参数
      : isFunctionExpression
        ? "(" + handler.value + ")($event)"
        : handler.value;
    /* istanbul ignore if */
    // 返回一个新的回调函数, 将修饰符对应的代码片段添加到原回调函数之前执行
    return "function($event){" + code + handlerCode + "}";
  }
}
```

接下来, vue 实例会根据上述得到的`code`进行渲染, 具体调用过程如下:

`vm._update() -> vm.__patch__() -> createElm -> invokeCreateHooks(通过cbs.create) -> updateDOMListeners -> add$1`

其实`add$1`是事件绑定的核心函数

```javascript
function add$1(event, handler, once$$1, capture, passive) {
  // 用withMacroTask对handler进行处理, 使得handler执行过程中触发的状态变化都保存到macrotask队列中,
  // 详见https://juejin.im/post/5a1af88f5188254a701ec230
  handler = withMacroTask(handler);
  if (once$$1) {
    handler = createOnceHandler(handler, event, capture);
  }
  target$1.addEventListener(
    event,
    handler,
    supportsPassive ? { capture: capture, passive: passive } : capture
  );
}

/**
 * Wrap a function so that if any code inside triggers state change,
 * the changes are queued using a Task instead of a MicroTask.
 */
function withMacroTask(fn) {
  return (
    fn._withTask ||
    (fn._withTask = function() {
      useMacroTask = true;
      var res = fn.apply(null, arguments);
      useMacroTask = false;
      return res;
    })
  );
}
```

## 组件上的事件

组件上的事件主要有两类: 原生事件和自定义事件.
在组件根元素上直接监听一个原生事件, 可以使用`.native`修饰符.

页面代码如下:

```html
<div id="app">
  <my-component @click.native="nativeclick" @receive="receive">
  </my-component>
</div>
<script src="./vue.js"></script>
<script>
  Vue.component('my-component', {
    name: 'my-component',
    template: '<div>component. <div @click.stop="send">click me</div></div>',
    methods: {
      send() {
        this.$emit('receive', 'msg from component')
      }
    }
  })
  new Vue({
    el: '#app',
    methods: {
      nativeclick() {
        console.log('nativeclick')
      },
      receive(msg) {
        console.log('receive:', msg)
      }
    }
  })
</script>
```

组件的编译过程与普通 html 元素一样:

`$mount -> compileToFunctions -> compile -> baseCompile -> var ast = parse(template.trim(), options) -> var code = generate(ast, options) -> genElement -> genData$2 -> genHandler`

得到的`ast`语法树如下:

<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/vue%E4%BA%8B%E4%BB%B6%E6%9C%BA%E5%88%B6-%E7%BB%84%E4%BB%B6ast.jpg" align=center />

可以看到在组件上监听的原生 click 事件被解析到`nativeEvents`对象中, 而监听组件自定义的`receive`事件则被解析到`events`对象中了.

最后得到的渲染代码`code`为:

`_c('div',{attrs:{"id":"app"}},[_c('my-component',{on:{"receive":receive},nativeOn:{"click":function($event){nativeclick($event)}}})],1)`

接下来会根据`code`进行渲染, 调用过程如下:

`vm._update() -> vm.__patch__() -> createElm`

因为`my-component`是一个组件, 所以创建该组件时会调用`createComponent`方法返回一个 vnode

```javascript
function createComponent(Ctor, data, context, children, tag) {
  // ...

  /*
  data: {
    nativeOn:{
      click: function
    },
    on: {
      receive: function
    }
  }
  */

  // 将自定义的事件缓存到listeners, 以此与普通的DOM事件做区分
  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  var listeners = data.on;

  // 将真正的原生事件保存到data.on
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn;

  // ...

  // return a placeholder vnode
  var name = Ctor.options.name || tag;
  var vnode = new VNode(
    "vue-component-" + Ctor.cid + (name ? "-" + name : ""),
    data,
    undefined,
    undefined,
    undefined,
    context,
    {
      Ctor: Ctor,
      propsData: propsData,
      listeners: listeners,
      tag: tag,
      children: children
    },
    asyncFactory
  );

  return vnode;
}
```

## 参考资料

- [vue 源码解析－事件机制](https://segmentfault.com/a/1190000009750348)
