# vue diff算法

vue 2.0 的diff算法实现位于core/vdom/patch.js中, 该算法来源于[snabbdom](https://github.com/snabbdom/snabbdom), 复杂度为O(n).

了解diff算法之前首先要了解virtual dom, virtual dom就是用一个js对象去描述一个dom节点, 具体可参考[vue 虚拟 DOM 原理](https://github.com/tzstone/blog/blob/master/vue/vue%E8%99%9A%E6%8B%9FDOM%E5%8E%9F%E7%90%86.md).

## diff过程

vue的 diff 只会对同层级的vnode进行对比, 不会进行跨层级比较.

![diff](https://github.com/tzstone/MarkdownPhotos/raw/master/vue-diff.png)

diff的过程就是调用 `patch` 函数不断去修改真实的dom. patch方法可以将vnode转换成真实dom并渲染出来:

![虚拟DOM](https://github.com/tzstone/MarkdownPhotos/raw/master/virtual_dom.png)

```javascript
// patch.js
function patch(oldVnode, vnode, ...) {
  // 判断新旧vnode是否值得对比
  if (sameVnode(oldVnode, vnode)) {
    // 进行子节点对比
    patchVnode(oldVnode, vnode, ...)
  } else {
    // 不值得对比
    // replacing existing element
    const oldElm = oldVnode.elm
    const parentElm = nodeOps.parentNode(oldElm)

    // 创建并插入真实的dom, 将真实dom赋值给vnode.elm
    createElm(
      vnode,
      insertedVnodeQueue,
      oldElm._leaveCb ? null : parentElm,
      nodeOps.nextSibling(oldElm)
    )

    // 删除旧节点
    removeVnodes([oldVnode], 0, 0)
  }
  // 返回更新后的真实dom, 在外层由vm.$el接收, 相当于patch后vm.$el更新为真实dom
  return vnode.elm
}

// 通过key, tag, data等判断两个vnode是否值得对比
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

patch函数有两个参数，`vnode`和`oldVnode`，也就是新旧两个虚拟节点。在这之前，我们先了解完整的vnode都有什么属性，举个一个简单的例子:

```javascript
// body下的 <div id="v" class="classA"><div> 对应的 oldVnode 就是

{
  el:  div  //对真实的节点的引用，本例中就是document.querySelector('#id.classA')
  tagName: 'DIV',   //节点的标签
  sel: 'div#v.classA'  //节点的选择器
  data: null,       // 一个存储节点属性的对象，对应节点的el[prop]属性，例如onclick , style
  children: [], //存储子节点的数组，每个子节点也是vnode结构
  text: null,    //如果是文本节点，对应文本节点的textContent，否则为null
}
```

el属性引用的是此 virtual dom对应的真实dom，`patch`的vnode参数的 `el` 最初是`null`，因为patch之前它还没有对应的真实dom.

patch的过程为:

1. 通过新旧vnode的key, tag, data等(`sameVnode(oldVnode, vnode)`)判断新旧vnode是不是值得对比, 如果是则调用`patchVnode(oldVnode, vnode)`
2. 如果不值得对比, 则会根据新vnode创建真实的dom并插入到父节点中, 并将真实dom赋值给vnode.elm, 然后删除旧节点
3. patch方法返回vnode.elm(即真实dom), 由于update时外层的调用为`vm.$el = vm.__patch__(prevVnode, vnode)`, 即此时 `vm.$el` 被更新为真实dom

### patchVnode

当新旧vnode值得对比时, 会调用`patchVnode`函数:

```javascript
function patchVnode (oldVnode, vnode, ...) {
  // 引用一致, 不需要修改
  if (oldVnode === vnode) return

  // 让vnode.elm引用到真实dom, 当elm改变时, vnode.elm会同步变化
  const elm = vnode.elm = oldVnode.elm
  // 新旧子vnode
  const oldCh = oldVnode.children
  const ch = vnode.children

  // 新节点非文本节点
  if (isUndef(vnode.text)) {
    // 新旧节点都有子节点
    if (isDef(oldCh) && isDef(ch)) {
      // 新旧子节点不一致, 进行子节点对比
      if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
    } 
    // 只有新节点有子节点
    else if (isDef(ch)) {
      if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
      // 添加子节点, elm已经引用了旧vnode的dom, 会直接在旧dom上添加子节点
      addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
    } 
    // 老节点有子节点, 新节点没有子节点, 直接删除老节点
    else if (isDef(oldCh)) {
      removeVnodes(oldCh, 0, oldCh.length - 1)
    } 
    // 新旧节点都没有子节点, 且老节点是文本节点
    else if (isDef(oldVnode.text)) {
      nodeOps.setTextContent(elm, '')
    }
  }
  // 新节点是文本节点, 新旧节点文本不相等
  else if (oldVnode.text !== vnode.text) {
    nodeOps.setTextContent(elm, vnode.text)
  }
}
```

节点对比的几种情况:

1. `oldVnode === vnode`: 引用一致, 不需要对比
2. `isDef(vnode.text) && oldVnode.text !== vnode.text`: 新节点是文本节点且新旧节点文本不一致, 通过`nodeOps.setTextContent(elm, vnode.text)`修改文本
3. `isDef(oldCh) && isDef(ch) && oldCh !== ch`: 存在新旧子节点且子节点不一样, 调用`updateChildren`进行子节点对比
4. `else if (isDef(ch))`: 新节点有子节点, 旧节点没有子节点, 直接添加子节点, 会在旧的dom上添加
5. `else if (isDef(oldCh))`: 旧节点有子节点, 新节点没有子节点, 直接删除旧节点

### updateChildren

```javascript
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  let oldStartIdx = 0
  let newStartIdx = 0
  let oldEndIdx = oldCh.length - 1
  let oldStartVnode = oldCh[0]
  let oldEndVnode = oldCh[oldEndIdx]
  let newEndIdx = newCh.length - 1
  let newStartVnode = newCh[0]
  let newEndVnode = newCh[newEndIdx]
  let oldKeyToIdx, idxInOld, vnodeToMove, refElm

  // removeOnly is a special flag used only by <transition-group>
  // to ensure removed elements stay in correct relative positions
  // during leaving transitions
  const canMove = !removeOnly

  if (process.env.NODE_ENV !== 'production') {
    checkDuplicateKeys(newCh)
  }
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    // 判断空节点
    if (isUndef(oldStartVnode)) {
      oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
    } 
    // 判断空节点
    else if (isUndef(oldEndVnode)) {
      oldEndVnode = oldCh[--oldEndIdx]
    } 
    // 旧首节点和新首节点是否值得对比, true则不需要移动dom
    else if (sameVnode(oldStartVnode, newStartVnode)) {
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      oldStartVnode = oldCh[++oldStartIdx]
      newStartVnode = newCh[++newStartIdx]
    } 
    // 旧尾节点和新尾节点是否值得对比, true则不需要移动dom
    else if (sameVnode(oldEndVnode, newEndVnode)) {
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      oldEndVnode = oldCh[--oldEndIdx]
      newEndVnode = newCh[--newEndIdx]
    } 
    // 旧首节点和新尾节点是否值得对比, 即是否是将旧首节点移到尾部
    else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      // 移动dom
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
      oldStartVnode = oldCh[++oldStartIdx]
      newEndVnode = newCh[--newEndIdx]
    } 
    // 旧尾节点和新首节点是否值得对比, 即是否是将旧尾节点移到首部
    else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      // 移动dom
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
      oldEndVnode = oldCh[--oldEndIdx]
      newStartVnode = newCh[++newStartIdx]
    } 
    else {
      // 返回旧子节点组的key到index的映射
      if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
      idxInOld = isDef(newStartVnode.key)
        // 如果新节点设置了key, 就查找旧节点是否存在相同的key
        ? oldKeyToIdx[newStartVnode.key] 
        // 新节点没有设置key, 遍历旧节点检查是否存在值得对比的vnode(sameVode)
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)

      // 新节点在旧节点中不存在, 则创建dom, 插入父节点中
      if (isUndef(idxInOld)) { // New element
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
      } 
      // 新节点在旧节点组中存在
      else {
        vnodeToMove = oldCh[idxInOld]
        // 新节点与旧节点组中匹配到的节点是否值得对比, 这里再次检查是为了避免不同元素设置了相同的key
        if (sameVnode(vnodeToMove, newStartVnode)) {
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
          oldCh[idxInOld] = undefined
          // 移动dom
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
        } else {
          // 不同元素设置了相同的key, 视为新元素
          // same key but different element. treat as new element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        }
      }
      newStartVnode = newCh[++newStartIdx]
    }
  }

  // oldCh先遍历完(此时newCh可能刚好也遍历完), 此时newStartIdx到newEndIdx之间的vnode都是新增的
  if (oldStartIdx > oldEndIdx) {
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
    // 会调用insertBefore插入dom
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
  } 
  // newCh先遍历完, 此时oldStartIdx到oldEndIdx之间的vnode在新子节点中已经不存在, 可以删除
  else if (newStartIdx > newEndIdx) {
    removeVnodes(oldCh, oldStartIdx, oldEndIdx)
  }
}

function createKeyToOldIdx (children, beginIdx, endIdx) {
  let i, key
  const map = {}
  for (i = beginIdx; i <= endIdx; ++i) {
    key = children[i].key
    if (isDef(key)) map[key] = i
  }
  return map
}  

function findIdxInOld (node, oldCh, start, end) {
  for (let i = start; i < end; i++) {
    const c = oldCh[i]
    if (isDef(c) && sameVnode(node, c)) return i
  }
}

function addVnodes (parentElm, refElm, vnodes, startIdx, endIdx, insertedVnodeQueue) {
  for (; startIdx <= endIdx; ++startIdx) {
    createElm(vnodes[startIdx], insertedVnodeQueue, parentElm, refElm, false, vnodes, startIdx)
  }
}
```

子节点的`对比过程`如下:

`oldCh`和`newCh`各有两个变量`startIdx`, `endIdx`标识数组的首尾, 对比的过程中逐渐往中间靠拢, 一旦`startIdx>endIdx`, 就表明`oldCh`和`newCh`中至少有一个已经遍历完了, 就会结束对比.

![diff](https://github.com/tzstone/MarkdownPhotos/raw/master/vue-diff-1.png)

1. `if (sameVnode(oldStartVnode, newStartVnode))`: 旧首节点和新首节点是否值得对比, 为true则不需要对dom进行移动
2. `else if (sameVnode(oldEndVnode, newEndVnode))`: 旧尾节点和新尾节点是否值得对比, 为true则不需要对dom进行移动
3. `else if (sameVnode(oldStartVnode, newEndVnode))`: 旧首节点和新尾节点是否值得对比, 即是否是将旧首节点移到尾部, 假设startIdx遍历到1:
  ![diff](https://github.com/tzstone/MarkdownPhotos/raw/master/vue-diff-move-end.png)
4. `else if (sameVnode(oldEndVnode, newStartVnode))`: 旧尾节点和新首节点是否值得对比, 即是否是将旧尾节点移到首部, 假设startIdx遍历到1:
  ![diff](https://github.com/tzstone/MarkdownPhotos/raw/master/vue-diff-move-start.png)
5. 如果新首节点设置了key, 则查找旧节点组是否存在相同key的节点; 如果没有设置key, 则遍历旧节点组, 调用`sameVode`方法判断旧节点组中是否存在值得对比的节点
    1. 如果新首节点存在于旧节点组中并且是`sameVode`, 则会移动dom的位置
    2. 否则创建dom, 并插入父节点中

`对比结束`时，分为两种情况:

1. `oldStartIdx > oldEndIdx`，可以认为`oldCh`先遍历完(此时newCh可能也刚好遍历完). 此时`newStartIdx`和`newEndIdx`之间的vnode是新增的，调用`addVnodes`，会通过`insertBefore`将对应的dom插入到`newCh[newEndIdx + 1].elm(或者null)`之前. 
   
    注意: `insertBefore`时, 如果新的节点已在文档中存在, 会先从文档中`移除`再插入.

    ![diff](https://github.com/tzstone/MarkdownPhotos/raw/master/vue-diff-end-1.png)

2. `newStartIdx > newEndIdx`，可以认为`newCh`先遍历完。此时`oldStartIdx`和`oldEndIdx`之间的vnode在新的子节点里已经不存在了，调用`removeVnodes`将它们从dom里删除

    ![diff](https://github.com/tzstone/MarkdownPhotos/raw/master/vue-diff-end-2.png)

参考资料:

[解析vue2.0的diff算法](https://segmentfault.com/a/1190000008782928)