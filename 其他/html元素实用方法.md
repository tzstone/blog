# html 元素实用方法合集

- table

原生的 table 元素有很便捷的创建方法:

```javascript
const tableEl = document.querySelector("table");
const headRow = tableEl.createTHead().insertRow();
headRow.insertCell().textContent = "Make";
headRow.insertCell().textContent = "Model";
headRow.insertCell().textContent = "Color";

const newRow = tableEl.insertRow();
newRow.insertCell().textContent = "Yes";
newRow.insertCell().textContent = "No";
newRow.insertCell().textContent = "Thank you";
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
  如果当前元素完全在浏览器可视区域内, 则不会滚动, 否则滚动到浏览器可视区域的相应位置. 该方法是`scrollIntoView`的变体, 接收一个`Boolean`类型参数, `有兼容性问题`.

  - `Boolean`
    - `true`: 默认参数, 元素将滚动到其可滚动的祖先元素的可视区域的中间位置.
    - `false`: 元素将滚动到距离其可滚动的祖先元素的可视区域最近的边缘位置(`nearest`), `top` or `buttom`.

  注意: 当元素不完全处于可视区域(某一部分被遮挡), 无论参数是`true`还是`false`, 调用`scrollIntoViewIfNeeded`都会发生滚动, 对齐的方式是`nearest`.

## 参考资料

- [你可能从未听说过这 15 个 HTML 元素方法！](https://mp.weixin.qq.com/s/QLoTIpVgq5IqqYDW4ZQMKg)
- [scrollIntoView 与 scrollIntoViewIfNeeded API 介绍](https://juejin.im/post/59d74afe5188257e8267b03f)
- [scrollIntoView block vs inline](https://stackoverflow.com/questions/48634459/scrollintoview-block-vs-inline)
