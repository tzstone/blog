# html 元素实用方法合集

- table 元素

  原生的 table 元素有很便捷的创建方法:

  ```javascript
  const tableEl = document.querySelector('table');
  const headRow = tableEl.createTHead().insertRow();
  headRow.insertCell().textContent = 'Make';
  headRow.insertCell().textContent = 'Model';
  headRow.insertCell().textContent = 'Color';

  const newRow = tableEl.insertRow();
  newRow.insertCell().textContent = 'Yes';
  newRow.insertCell().textContent = 'No';
  newRow.insertCell().textContent = 'Thank you';
  ```

- el.scrollIntoView(Boolean | Object)

  将当前元素滚动到浏览器的可视区域内, 接收一个`Boolean`类型或者`Object`类型的参数.

  - `Boolean`
    - `true`: 默认参数, 元素的顶部将与其可滚动的祖先元素的可视区域的顶部对齐. 等同于`Object`类型的`{block: "start", inline: "nearest"}`.
    - `false`: 元素的底部将与其可滚动的祖先元素的可视区域的底部对齐. 等同于`Object`类型的`{block: "end", inline: "nearest"}`.
  - `Object`
    - `behavior`: 定义过渡动画. `auto`, `instant`, `smooth`之一. 默认是`auto`. `auto`, `instant`都是立即滚动, `smooth`是平滑滚动(`有兼容性问题`).
    - `block`: 定义元素在可滚动的祖先元素的可视区域内的垂直对齐方式. `start`, `center`, `end`, `nearest`之一. 默认是`center`.
    - `inline`: 定义元素在可滚动的祖先元素的可视区域内的水平对齐方式. `start`, `center`, `end`, `nearest`之一. 默认是`center`.

- el.scrollIntoViewIfNeeded(Boolean)

  如果当前元素`完全处于`浏览器可视区域内, 则不会滚动, 否则滚动到浏览器可视区域的相应位置. 该方法是`scrollIntoView`的变体, 接收一个`Boolean`类型参数, `有兼容性问题`.

  - `Boolean`
    - `true`: 默认参数, 元素将滚动到其可滚动的祖先元素的可视区域的中间位置.
    - `false`: 元素将滚动到距离其可滚动的祖先元素的可视区域最近的边缘位置(`nearest`), `top` or `buttom`.

  注意: 当元素不完全处于可视区域(某一部分被遮挡), 无论参数是`true`还是`false`, 调用`scrollIntoViewIfNeeded`都会发生滚动, 对齐的方式是`nearest`.

- el.hidden

  可用来读取或设置元素隐藏, 类似`el.style.display='none'`.

  `isHidden = el.hidden;`

  `el.hidden = true | false;`

  注意: 如果设置了某个元素的`display`为`none`, 其`hidden`属性仍然为`false`.

- el.closest(selectors)

  返回当前元素匹配到的最近的祖先元素, 如果没匹配到, 则返回 null.

  `el.closest('.box')`

- el.getBoundingClientRect()

  返回当前元素的大小及其相对视窗的位置: `left`, `top`, `right`, `bottom`, `x`, `y`, `width`, `height`.

  除了`width`, `height`, 其他属性都是相对于视窗的 top-left 为坐标轴确定的.

  注意: 该方法会引起重绘

  <img src="https://github.com/tzstone/MarkdownPhotos/raw/master/getBoundingClientRect.gif" height=250 align=center />

  <img src="https://github.com/tzstone/MarkdownPhotos/raw/master/getBoundingClientRect-scroll.gif" height=250 align=center />

- el.matches(selectorString)

  returns `true` if the element would be selected by the specified selector string; otherwise, returns `false`.

  ```html
  <div class="wrap">
    <div class="box"></div>
  </div>
  <script>
    document.querySelector('.box').matches('.wrap .box'); // true
    document.querySelector('.box').matches('.box'); // true
  </script>
  ```

- el.insertAdjacentElement(position, element)

  inserts a given element node at a given position relative to the element it is invoked upon

  - `position`
    - `beforebegin`: Before the `el` itself.
    - `afterbegin`: Just inside the `el`, before its first child.
    - `beforeend`: Just inside the `el`, after its last child.
    - `afterend`: After the `el` itself.
  - `element`: The element to be inserted into the tree

  ```html
  <!-- before -->
  <div class="box">
    <span>first child</span>
    <span>secend child</span>
    <span>last child</span>
  </div>

  <!-- after box.insertAdjacentElement insert-->
  <span>beforebegin</span>
  <div class="box">
    <span>afterbegin</span>
    <span>first child</span>
    <span>secend child</span>
    <span>last child</span>
    <span>beforeend</span>
  </div>
  <span>afterend</span>
  ```

- node.contains(otherNode)

  returns a Boolean value indicating whether a node is a descendant of a given node

  element-ui 里 clickoutside 的自定义指令有类似用法:

  ```javascript
  const handler = function (e) {
    if (!el.contains(e.target)) {
      // ...
    }
  };
  ```

- dialog 元素

  `dialogEl.show()`, `dialogEl.hide()`方法可以控制 dialog 元素的显示/隐藏.

  `dialogEl.showModal()`能够将`dailog`元素显示在页面的顶层, 居中对齐, 无需指定`z-index`或添加灰色背景色(浏览器会默认添加背景色), 也不需要监听 esc 按键以关闭此窗口.

  注意:

  - 在`dialog`元素里内置`form`元素, 将其`method`设置为`dialog`, 当表单`submit`的时候, `dialog`会自动关闭.
  - 调用`show`方法时, `dialog`元素是按普通文档流的方式排列; 调用`showModal`方法时, `dialog`元素是相对视图垂直, 水平居中(类似`fixed`定位)

- form 表单

  - `reset`: 重置表单
  - `reportValidity`: 表单校验

## 参考资料

- [你可能从未听说过这 15 个 HTML 元素方法！](https://mp.weixin.qq.com/s/QLoTIpVgq5IqqYDW4ZQMKg)
- [scrollIntoView 与 scrollIntoViewIfNeeded API 介绍](https://juejin.im/post/59d74afe5188257e8267b03f)
- [scrollIntoView block vs inline](https://stackoverflow.com/questions/48634459/scrollintoview-block-vs-inline)
