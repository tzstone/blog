# vue 虚拟 DOM 原理

## 虚拟 DOM 的定义

在浏览器中, 真正的 DOM 元素是非常复杂庞大的，当我们频繁的去做 DOM 更新时，会产生一定的性能问题。

而 Virtual DOM 就是用一个原生的 `JS 对象`去描述一个 DOM 节点，所以它比创建一个 DOM 的代价要小很多。在 Vue.js 中，Virtual DOM 是用 VNode 这么一个 Class 去描述的。Vue.js 中 Virtual DOM 是借鉴了开源库 [snabbdom](https://github.com/snabbdom/snabbdom) 的实现。

VNode 是对真实 DOM 的一种`抽象描述`，它的核心定义无非就几个关键属性，标签名、数据、子节点、键值等，其它属性都是用来扩展 VNode 的灵活性以及实现一些特殊 feature 的。由于 VNode 只是用来映射到真实 DOM 的渲染，不需要包含操作 DOM 的方法，因此它是非常`轻量`和`简单`的。

## 虚拟 DOM 的创建/更新

Virtual DOM 除了它的数据结构的定义，映射到真实的 DOM 实际上要经历 VNode 的 create、diff、patch 等过程。如下图:
![虚拟DOM](https://github.com/tzstone/MarkdownPhotos/raw/master/virtual_dom.png)

一个真实的例子:
![虚拟DOM](https://github.com/tzstone/MarkdownPhotos/raw/master/virtual_dom_1.png)

### create

Vue.js 在 mounted 方法中将 template 编译成渲染函数(render), 渲染函数通过调用 `createElement` 方法返回一个 `VNode`。createElement 方法本身返回一个 VNode, 同时也会将传入的 children 转换为 VNode 类型。这样每个 VNode 有 children，children 每个元素也是一个 VNode，形成了一个 `VNode Tree`，它很好地描述了我们的 `DOM Tree`。

### patch

将 VNode 渲染成一个真实的 DOM 并渲染出来，这个过程是通过 vm.\_update 完成的。\_update 的核心就是调用 vm.\_\_patch\_\_ 方法，这个方法实际上在不同的平台，比如 web 和 weex 上的定义是不一样的。甚至在 web 平台上，是否是服务端渲染也会对这个方法产生影响。因为在服务端渲染中，没有真实的浏览器 DOM 环境，所以不需要把 VNode 最终转换成 DOM，因此是一个空函数，而在浏览器端渲染中，它指向了 patch 方法。

```javascript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this;
  const prevVnode = vm._vnode;
  vm._vnode = vnode;
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */);
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode);
  }
  // ...
};

Vue.prototype.__patch__ = inBrowser ? patch : noop;
```

patch 方法中最终会调用 createElm 方法。createElm 的作用是通过虚拟节点创建真实的 DOM 并插入到它的父节点中。createElm 会为传入的 vnode 创建一个占位符元素, 然后遍历子虚拟节点, 递归调用 createElm(深度优先), 将生成的子节点插入父节点中。

```javascript
/* patch */
// replacing existing element
const oldElm = oldVnode.elm;
const parentElm = nodeOps.parentNode(oldElm);

// create new node
createElm(
  vnode,
  insertedVnodeQueue,
  // extremely rare edge case: do not insert if old element is in a
  // leaving transition. Only happens when combining transition +
  // keep-alive + HOCs. (#4590)
  oldElm._leaveCb ? null : parentElm,
  nodeOps.nextSibling(oldElm),
);

/* createElm */
// 占位符元素
vnode.elm = vnode.ns ? nodeOps.createElementNS(vnode.ns, tag) : nodeOps.createElement(tag, vnode);
// 创建子节点, vnode.elm为子节点的父容器占位符
createChildren(vnode, children, insertedVnodeQueue);
// 把占位符元素插入父节点
insert(parentElm, vnode.elm, refElm);

// 创建子节点
function createChildren(vnode, children, insertedVnodeQueue) {
  if (Array.isArray(children)) {
    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(children);
    }
    for (let i = 0; i < children.length; ++i) {
      // 递归调用createElm, 会把vnode.elm作为父容器的DOM节点占位符传入
      createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i);
    }
  } else if (isPrimitive(vnode.text)) {
    nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)));
  }
}
```

### diff

patch 方法有两个调用时机:

1. 首次渲染 patch(container, vnode): 将 vnode 渲染成真正的 DOM 然后插入到容器里面。
2. 数据更新 patch(oldVnode,newVnode): 将新旧的 vnode 进行对比，然后将这两者之间差异应用到所构建的真正的 DOM 树上(数据更新时会重新执行一遍渲染函数, 重新收集依赖, 并得到新的 vnode)。

在 Vue 中, 数据更新本质上是重新渲染了整个视图(重新执行渲染函数), 然后用新的视图替换掉旧的视图, 但通过 vnode 的 diff 运算来避免整个 DOM 树的变更, 而是只更新发生变化的节点。

Vue 的 diff 算法是基于 snabbdom 改造过来的，仅在`同级`的 vnode 间做 diff，递归地进行同级 vnode 的 diff，最终实现整个 DOM 树的更新。diff 算法有以下步骤:

1. 调用 sameVnode 方法判断是不是相同的 vnode
2. 如果新旧 vnode 相同, 则调用 patchVNode 方法, 把新的 vnode patch 到旧的 vnode 上
3. 如果新旧 vnode 不同, 那么:
   1. 创建新节点: 以当前旧节点为参考节点, 创建新的节点, 并插入到 DOM 中
   2. 更新父的占位符节点: 找到当前 vnode 的父的占位符节点，先执行各个 module 的 destroy 的钩子函数，如果当前占位符是一个可挂载的节点，则执行 module 的 create 钩子函数。
   3. 删除旧节点: 把 oldVnode 从当前 DOM 树中删除，如果父节点存在，则执行 removeVnodes 方法

```code
// 判断是否是相同的 vnode
function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

## 虚拟 DOM 的优势

虚拟 DOM 相当于在真实的 DOM 上加了一层抽象(或者说, 对真实的 DOM 操作加了一个拦截), 同时, 它也是一个 `JS 对象`, 可以接受 `Parser 解析转化`, 这意味着我们可以在`编译阶段`做很多事情, 比如：xss 攻击（\$\$typeof）、合并 DOM 操作、跨平台 等等。虚拟 DOM 有如下优点:

- 渲染性能提升

  频繁的 DOM 操作是很昂贵的, 虚拟 DOM 一方面可以将多次 DOM 操作进行整合, 另一方面可以通过 diff 算法计算出真正需要更新的节点, 最大限度减少 DOM 操作, 提高性能。

  不过, 相对于直接操作真实的 DOM, 虚拟 DOM 操作始终会多出 diff 运算这一步骤, 另一方面, 现代浏览器也非常"聪明", 对 DOM 操作有各种优化, 所以提升渲染性能并不是使用虚拟 DOM 的主要考量。

- 跨平台

  虚拟 DOM 只是真实 DOM 的抽象, 不依赖于宿主环境, 只需要平台能运行 JS 脚本即可, 天然具有跨平台的能力, 可以运行在浏览器, Weex, Node 等多种平台下。

## 虚拟 DOM 的实现

(以下实现来自[深度剖析：如何实现一个 Virtual DOM 算法](https://segmentfault.com/a/1190000004029168))

虚拟 DOM 的简单算法实现, 主要有以下步骤:

1. 用 JavaScript 对象结构表示 DOM 树的结构；然后用这个树构建一个真正的 DOM 树，插到文档当中
2. 当状态变更的时候，重新构造一棵新的对象树。然后用新的树和旧的树进行比较，记录两棵树差异
3. 把 2 所记录的差异应用到步骤 1 所构建的真正的 DOM 树上，视图就更新了

下面是具体的实现算法:

`步骤一`: 使用 javascript 对象表示 DOM 节点的节点类型, 属性和子节点:

```javascript
function Element(tagName, props, children) {
  this.tagName = tagName;
  this.props = props;
  this.children = children;
}

module.exports = function (tagName, props, children) {
  return new Element(tagName, props, children);
};
```

```html
<ul id="list">
  <li class="item">Item 1</li>
  <li class="item">Item 2</li>
  <li class="item">Item 3</li>
</ul>
```

上面的 DOM 结构可以表示为:

```code
 var el = require('./element')

 var ul = el('ul', {id: 'list'}, [
   el('li', {class: 'item'}, ['Item 1']),
   el('li', {class: 'item'}, ['Item 2']),
   el('li', {class: 'item'}, ['Item 3'])
 ])
```

将 javascript 对象转换为真实的 DOM:

```javascript
// 根据tagName构建一个真正的DOM节点，然后设置这个节点的属性，最后递归地把自己的子节点也构建起来
Element.prototype.render = function () {
  var el = document.createElement(this.tagName); // 根据tagName构建
  var props = this.props;

  for (var propName in props) {
    // 设置节点的DOM属性
    var propValue = props[propName];
    el.setAttribute(propName, propValue);
  }

  var children = this.children || [];

  children.forEach(function (child) {
    var childEl =
      child instanceof Element
        ? child.render() // 如果子节点也是虚拟DOM，递归构建DOM节点
        : document.createTextNode(child); // 如果字符串，只构建文本节点
    el.appendChild(childEl);
  });

  return el;
};
```

将生成的 DOM 节点插入文档, 这样 body 就有了真正的 \<ul\> DOM 结构:

```javascript
var ulRoot = ul.render();
document.body.appendChild(ulRoot);
```

`步骤二`: 比较两棵虚拟 DOM 树的差异(即 diff 算法)

两个树的完全的 diff 算法是一个时间复杂度为 O(n^3) 的问题。但是在前端当中，你很少会跨越层级地移动 DOM 元素。所以 Virtual DOM 只会对同一个层级的元素进行对比：

![diff](https://github.com/tzstone/MarkdownPhotos/raw/master/compare-in-level.png)

上面的 div 只会和同一层级的 div 对比，第二层级的只会跟第二层级对比。这样算法复杂度就可以达到 O(n)。

1. 深度优先遍历，`记录差异`

   在实际的代码中，会对新旧两棵树进行一个深度优先的遍历，这样每个节点都会有一个`唯一的标记`：

   ![diff](https://github.com/tzstone/MarkdownPhotos/raw/master/dfs-walk.png)

   在深度优先遍历的时候，每遍历到一个节点就把该节点和新的的树进行对比。如果有差异的话就记录到一个对象里面。

   ```javascript
   // diff 函数，对比两棵树
   function diff (oldTree, newTree) {
     var index = 0 // 当前节点的标志
     var patches = {} // 用来记录每个节点差异的对象
     dfsWalk(oldTree, newTree, index, patches)
     return patches
   }

   // 对两棵树进行深度优先遍历
   function dfsWalk (oldNode, newNode, index, patches) {
     // 对比oldNode和newNode的不同，记录下来
     patches[index] = [...]

     diffChildren(oldNode.children, newNode.children, index, patches)
   }

   // 遍历子节点
   function diffChildren (oldChildren, newChildren, index, patches) {
     var leftNode = null
     var currentNodeIndex = index
     oldChildren.forEach(function (child, i) {
       var newChild = newChildren[i]
       currentNodeIndex = (leftNode && leftNode.count) // 计算节点的标识
         ? currentNodeIndex + leftNode.count + 1
         : currentNodeIndex + 1
       dfsWalk(child, newChild, currentNodeIndex, patches) // 深度遍历子节点
       leftNode = child
     })
   }
   ```

   例如，上面的 div 和新的 div 有差异，当前的标记是 0，那么：

   ```javascript
   patches[0] = [{difference}, {difference}, ...] // 用数组存储新旧节点的不同
   ```

   同理 p 是 patches[1]，ul 是 patches[3]，类推。

2. 差异类型

   上面说的节点的差异指的是什么呢？对 DOM 操作可能会：

   - 替换掉原来的节点，例如把上面的 div 换成了 section
   - 移动、删除、新增子节点，例如上面 div 的子节点，把 p 和 ul 顺序互换
   - 修改了节点的属性
   - 对于文本节点，文本内容可能会改变。例如修改上面的文本节点 2 内容为 Virtual DOM 2。

   所以我们定义了几种差异类型：

   ```javascript
   var REPLACE = 0;
   var REORDER = 1;
   var PROPS = 2;
   var TEXT = 3;
   ```

   对于节点替换，很简单。判断新旧节点的 tagName 和是不是一样的，如果不一样的说明需要替换掉。如 div 换成 section，就记录下：

   ```javascript
   patches[0] = [
     {
       type: REPALCE,
       node: newNode, // el('section', props, children)
     },
   ];
   ```

   如果给 div 新增了属性 id 为 container，就记录下：

   ```javascript
   patches[0] = [
     {
       type: REPALCE,
       node: newNode, // el('section', props, children)
     },
     {
       type: PROPS,
       props: {
         id: 'container',
       },
     },
   ];
   ```

   如果是文本节点，如上面的文本节点 2，就记录下：

   ```javascript
   patches[2] = [
     {
       type: TEXT,
       content: 'Virtual DOM2',
     },
   ];
   ```

   那如果把我 div 的子节点重新排序呢？例如 p, ul, div 的顺序换成了 div, p, ul。这个该怎么对比？如果按照同层级进行顺序对比的话，它们都会被替换掉。如 p 和 div 的 tagName 不同，p 会被 div 所替代。最终，三个节点都会被替换，这样 DOM 开销就非常大。而实际上是不需要替换节点，而只需要经过节点移动就可以达到，我们只需知道怎么进行移动。这牵涉到两个列表的对比算法。

3. 列表对比算法

   假设现在可以英文字母唯一地标识每一个子节点：

   旧的节点顺序：

   ```code
     a b c d e f g h i
   ```

   现在对节点进行了删除、插入、移动的操作。新增 j 节点，删除 e 节点，移动 h 节点：

   新的节点顺序：

   ```code
   a b c h d f g h i j
   ```

   现在知道了新旧的顺序，求最小的插入、删除操作（移动可以看成是删除和插入操作的结合）。这个问题抽象出来其实是字符串的最小编辑距离问题（[Edition Distance](https://en.wikipedia.org/wiki/Edit_distance)），最常见的解决算法是 [Levenshtein Distance](https://en.wikipedia.org/wiki/Levenshtein_distance)，通过动态规划求解，时间复杂度为 O(M \* N)。但是我们并不需要真的达到最小的操作，我们只需要优化一些比较常见的移动情况，牺牲一定 DOM 操作，让算法时间复杂度达到线性的（O(max(M, N))。具体算法细节比较多，这里不累述，有兴趣可以参考[代码](https://github.com/livoras/list-diff/blob/master/lib/diff.js)。

   我们能够获取到某个父节点的子节点的操作，就可以记录下来：

   ```javascript
   patches[0] = [{
     type: REORDER,
     moves: [{remove or insert}, {remove or insert}, ...]
   }]
   ```

   但是要注意的是，因为 tagName 是可重复的，不能用这个来进行对比。所以需要给子节点加上唯一标识 key，列表对比的时候，使用 key 进行对比，这样才能复用老的 DOM 树上的节点。

   这样，我们就可以通过深度优先遍历两棵树，每层的节点进行对比，记录下每个节点的差异了。完整 diff 算法代码可见 [diff.js](https://github.com/livoras/simple-virtual-dom/blob/master/lib/diff.js)。

`步骤三`：把差异应用到真正的 DOM 树上

因为步骤一所构建的 JavaScript 对象树和 render 出来真正的 DOM 树的信息、结构是一样的。所以我们可以对那棵 DOM 树也进行深度优先的遍历，遍历的时候从步骤二生成的 patches 对象中找出当前遍历的节点差异，然后进行 DOM 操作。

```javascript
function patch(node, patches) {
  var walker = { index: 0 };
  dfsWalk(node, walker, patches);
}

function dfsWalk(node, walker, patches) {
  var currentPatches = patches[walker.index]; // 从patches拿出当前节点的差异

  var len = node.childNodes ? node.childNodes.length : 0;
  for (var i = 0; i < len; i++) {
    // 深度遍历子节点
    var child = node.childNodes[i];
    walker.index++;
    dfsWalk(child, walker, patches);
  }

  if (currentPatches) {
    applyPatches(node, currentPatches); // 对当前节点进行DOM操作
  }
}
```

applyPatches，根据不同类型的差异对当前节点进行 DOM 操作：

```javascript
function applyPatches(node, currentPatches) {
  currentPatches.forEach(function (currentPatch) {
    switch (currentPatch.type) {
      case REPLACE:
        node.parentNode.replaceChild(currentPatch.node.render(), node);
        break;
      case REORDER:
        reorderChildren(node, currentPatch.moves);
        break;
      case PROPS:
        setProps(node, currentPatch.props);
        break;
      case TEXT:
        node.textContent = currentPatch.content;
        break;
      default:
        throw new Error('Unknown patch type ' + currentPatch.type);
    }
  });
}
```

完整代码可见 [patch.js](https://github.com/livoras/simple-virtual-dom/blob/master/lib/patch.js)。

参考资料

[Vue 技术揭秘 -- Virtual DOM](https://ustbhuangyi.github.io/vue-analysis/v2/data-driven/virtual-dom.html)

[Vue 技术揭秘 -- 组件更新](https://ustbhuangyi.github.io/vue-analysis/v2/reactive/component-update.html)

[浅谈 Vue 中的虚拟 DOM](https://github.com/zhengjiaqing/blog/issues/11)

[深度剖析：如何实现一个 Virtual DOM 算法](https://segmentfault.com/a/1190000004029168)

[React 性能优化-虚拟 Dom 原理浅析](https://www.jianshu.com/p/e131df377053?utm_source=oschina-app)

[Vue 为什么要用虚拟 DOM(Virtual DOM)](https://learnku.com/articles/50487)
