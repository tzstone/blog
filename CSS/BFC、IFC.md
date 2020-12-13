# BFC、IFC

## FC（Formatting Context）

它是 W3C CSS2.1 规范中的一个概念，定义的是页面中的一块渲染区域，并且有一套渲染规则，它决定了其子元素将如何定位，以及和其他元素的关系和相互作用。

普通流中的框(boxes)属于某一个格式化上下文，它可以是块(block)的或内联(inline)的，但不能同时出现。块级(block-level)框参与块格式化上下文(block formatting context, BFC)。内联级(inline-level)框参与内联格式化上下文(inline formatting context, IFC)。

以下排版过程来着"重学前端"课程:

### 排版过程: 格式化上下文(formatting context) + 框(boxes)/文字(charater) = 位置(positions)

当我们要把正常流中的一个框或者文字排版, 需要分成三种情况处理:

1. 当遇到块级框: 排入块级格式化上下文
2. 当遇到内联级框或者文字: 首先尝试排入内联级格式化上下文, 如果排不下, 那么创建一个行框, 先将行框排版(行框是块级, 所以到第一种情况), 行框会创建一个内联级格式化上下文
3. 遇到 float 框: 把框的顶部跟当前内联级格式化上下文上边缘对齐, 然后根据 float 的方向把框的对应边缘对到块级格式化上下文的边缘, 之后重排当前行框

### 块级元素([block-level elements](https://www.w3.org/TR/CSS2/visuren.html#block-level))

`块级元素`是源文档中可视格式为块（例如段落）的那些元素。 display 属性的以下值使元素具有块级：`block`，`list-item` 和 `table`。

块级框(boxes)是参与块格式化上下文的框。每个块级元素都会生成一个`主块级框`(principal block-level box)，它包含子代框和生成的内容，并且(主块级框)也是任何定位方案中涉及的框。一些块级元素可能会生成除主体框外的其他框：list-item 元素。这些附加的框是相对于主体框放置。

除了表框(table boxes)和被替换的元素(replaced elements)外，块级框也是块容器框(block container box)。`块容器框要么只包含块级框，要么建立内联格式化上下文(因此仅包含内联级框)。`并非所有的块容器框都是块级框：不可替换的内联块(inline block)和不可替换的表格单元格(table cell)是块容器，但不是块级框。块级框(block-level boxes)也是块容器(block containers), 称为块框(block boxes)。

如果一个块容器框内部有一个块级框, 将强制其内部只有块级框。

这三个术语“块级框(block-level box)”、“块容器框(block container box)”和“块框(block box)”有时被缩写为“块(block)”。

### 内联级元素([inline-level elements](https://www.w3.org/TR/CSS2/visuren.html#inline-level))

`内联级元素`是源文档中不构成新内容块的那些元素；内容按行分布（例如，段落中的强调文本片段，内联图像等）。 display 属性的以下值使元素处于内联级别(inline-level)：`inline`，`inline-table` 和 `inline-block`。内联级元素生成内联级框(inline-level boxes)，这些框参与内联格式化上下文。

内联框是内联级别的，它的内容参与到其包含(containing)内联格式化上下文中。一个 display 值为 inline 的未被替换的元素会生成一个内联框(inline box)。非内联框的内联级框(如替换的内联级(inline-level)元素、内联块(inline-block)元素和内联表(inline-table)元素)称为原子内联级框，因为它们作为单个不透明框参与内联格式上下文。

## BFC

块级格式化上下文（Block Formatting Context，BFC） 是 Web 页面可视化 CSS 渲染的一部分，是块盒子(block boxes)的布局过程发生的区域，也是浮动元素与其他元素交互的区域。

### [布局规则](https://www.w3.org/TR/CSS2/visuren.html#block-formatting)

浮动元素，绝对定位元素，不是块框(block boxes)的块容器(block containers)（如 inline-blocks, table-cells, and table-captions），以及 overflow 属性不是 visible 的块框(block boxes)（除非该值已传播(propagated)到 viewport）会为其内容创建新的块级格式化上下文。

在块级格式化上下文中，框(boxes)从包含块(containing block)的顶部开始一个接一个地垂直布置。 两个兄弟框之间的垂直距离由 margin 属性决定。 块级格式化上下文中相邻块级框(block-level boxes)之间的垂直边距折叠。

在块级格式化上下文中，每个框的左外边缘(left outer edge)与包含块的左边缘接触（对于 right-to-left formating，则右边缘接触）。即使存在浮动元素，也是如此（尽管框(box)的行框(line boxes)可能由于浮动元素而缩小），除非框建立了新的块级格式化上下文（在这种情况下，框本身可能会由于浮动元素而变窄）。

下列方式会创建块格式化上下文：

- 根元素（`<html>`）
- 浮动元素（元素的 `float` 不是 none）
- 绝对定位(Absolutely positioned)元素（元素的 `position` 为 absolute 或 fixed）
- 行内块元素（元素的 `display` 为 inline-block）
- 表格单元格（元素的 `display` 为 table-cell，HTML 表格单元格默认为该值）
- 表格标题（元素的 `display` 为 table-caption，HTML 表格标题默认为该值）
- 匿名表格单元格元素（元素的 `display` 为 table、table-row、 table-row-group、table-header-group、table-footer-group（分别是 HTML table、row、tbody、thead、tfoot 的默认属性）或 inline-table）
- `overflow` 值不为 visible 或 clip 的块元素
- `display` 值为 flow-root 的元素
- `contain` 值为 layout、content 或 paint 的元素
- 弹性元素（`display` 为 flex 或 inline-flex 元素的直接子元素）, 且它们本身不是 flex, grid, 或者 table 容器(container)
- 网格元素（`display` 为 grid 或 inline-grid 元素的直接子元素）, 且它们本身不是 flex, grid, 或者 table 容器(container)
- 多列容器（元素的 `column-count` 或 `column-width` 不为 auto，包括 column-count 为 1）
- `column-span` 为 all 的元素始终会创建一个新的 BFC，即使该元素没有包裹在一个多列容器中（[标准变更](https://github.com/w3c/csswg-drafts/commit/a8634b96900279916bd6c505fda88dda71d8ec51)，[Chrome bug](https://bugs.chromium.org/p/chromium/issues/detail?id=709362)）。

注意：Flex / Grid 容器（display：flex / grid / inline-flex / inline-grid）建立新的 Flex / Grid 格式化上下文，除了布局外与块格式化上下文相似。 在 flex / grid 容器中没有可用的浮动子元素，但排除外部浮动及阻止外边距折叠仍然有效。

### 应用

可以将 BFC 视为页面中的迷你布局(mini layout), 一旦元素创建了 BFC , 所有内容(包括浮动元素)都会被包含其中. BFC 会阻止它们逃脱并从框中穿出来. 因此 BFC 有如下特性：

- 包含内部浮动

  - 创建 BFC 的容器会将子浮动元素包含其中, 所以容器计算高度时, 会将子浮动元素的高度也计算进去. 由此解决了子浮动元素撑不开父元素高度的问题.

- 排除外部浮动

  - 在正常流中创建新 BFC 的元素不会环绕(wrap)任何浮动元素, [示例](https://codepen.io/pen/?&prefill_data_id=d59e2663-e252-43cf-8be3-ac02c0897399)
  - 可实现两列布局(浮动元素与 BFC 容器各一列), 这种布局方式的好处是不需要指定左右两列的宽度
  - flexbox 是实现多列布局更有效的方式

- 阻止外边距折叠(margin collapsing)

  - 创建了 BFC 的元素不会和它的普通流子元素发生外边距叠加(BFC 会将任何内容包含其中, 包括普通流子元素的边距)

  - 避免特殊情况下元素自身的 margin 重叠(下面外边距折叠发生情况的第 3 点的第 4 小点)

### [外边距折叠](https://www.w3.org/TR/CSS2/box.html#collapsing-margins)

在 CSS 中，两个或多个`相邻`的`普通流`(in flow)中的盒子（可能是父子元素，也可能是兄弟元素）在`垂直方向`上的外边距会发生叠加，这种形成的外边距称之为外边距折叠。

以下情况会发生外边距折叠:

- 都属于普通流的块级盒子(block-level boxes)且参与到相同的块级格式上下文中
- 没有被 `padding`、`border`、`clear` 和 `line box` 分隔开
- 都属于垂直相邻(vertically-adjacent)盒子边缘：

  - 盒子的 `top margin` 和它第一个普通流子元素的 `top margin`
  - 盒子的 `bottom margin` 和它下一个普通流兄弟元素的 `top margin`
  - 最后一个普通流子元素的 `bottom margin` 和它父元素的 `bottom margin`, 如果它父元素有被计算为 `auto` 的 `height`
  - 盒子的 `top margin` 和 `bottom margin`，且没有创建一个新的块级格式上下文，且有被计算为 `0` 的 `min-height`，被计算为 0 或 auto 的 `height`，且没有普通流子元素(相当于盒子实际高度为 0, 盒子自身的 margin-top, margin-bottom 发生了重叠)

[如何阻止外边距折叠](https://www.w3.org/TR/CSS2/box.html#collapsing-margins):

- 浮动元素不会与任何元素发生叠加, 也包括其普通流子元素
- 创建了 BFC 的元素不会和它的普通流子元素发生外边距叠加
- 绝对定位(absolute, fixed 定位)元素不会发生外边距叠加，包括与其普通流子元素
- inline-block 元素不会发生外边距叠加，包括与其普通流子元素
- 普通流块级元素的 margin-bottom 通常和它下一个相邻的普通流块级元素的 margin-top 叠加，除非相邻的兄弟元素 [clear](https://www.w3.org/TR/CSS2/visuren.html#clearance)
- 普通流块级元素（没有 border-top、padding-top）的 margin-top 和它的第一个普通流块级子元素（没有 clear）的 margin-top 发生叠加
- 普通流块级元素（height 为 auto, 且 min-height 为 0、没有 border-bottom、padding-bottom）的 margin-bottom 和它的最后一个普通流块级子元素（bottom margin 没有与另一个 clear 的 top margin 发生叠加）的 margin-bottom 发生叠加
- 如果一个元素的 min-height 为 0、没有 border(top、bottom)、没有 padding(top、bottom)、height 为 0 或者 auto、不包含一个 line box，那么它所有的普通流子元素的外边距会发生叠加

### 普通流(in flow)

只要不是`float`、`absolutely positioned`(position 为 absolute 或 fixed)和`root element`时就是 in flow

## IFC

### [布局规则](https://www.w3.org/TR/CSS2/visuren.html#inline-formatting)

在内联格式化上下文(Inline Formatting Context, IFC)，框(boxes)从包含块(containig block)的顶部开始一个接一个地水平排列。水平方向上的 margin，border 和 padding 在框之间得到保留。这些框在垂直方向上可以以不同的方式对齐：它们的顶部或底部对齐，或根据其中文字的基线对齐。包含那些框的矩形区域，会形成一行，叫做行框(line box)。

行框的宽度由包含块和浮动元素的存在决定。行框的高度由 line height 的[计算规则](https://www.w3.org/TR/CSS2/visudet.html#line-height)决定。

一个行框的高度总是足以容纳它所包含的所有框。但是，它可能比它所包含的最高的框还要高(例如，框对齐(aligned)使基线(baselines)对齐(line up))。当框 B 的高度小于包含它的行框的高度时，B 在行框内的垂直对齐方式由 `vertical-align` 属性决定。当多个内联级框(inline-level boxes)不能水平地容纳在单个行框内时，它们将分布在两个或多个垂直堆叠(vertically-stacked)的行框中。因此，段落是行框的垂直堆叠。行框堆叠时没有垂直间隔（除非在其他地方指定），并且它们永远不会重叠。

参考资料

[Block formatting context](https://developer.mozilla.org/en-US/docs/Web/Guide/CSS/Block_formatting_context)

[Inline formatting contexts](https://www.w3.org/TR/CSS2/visuren.html#inline-formatting)

[Understanding CSS Layout And The Block Formatting Context
](https://www.smashingmagazine.com/2017/12/understanding-css-layout-block-formatting-context/)

[how-does-the-css-block-formatting-context-work](https://stackoverflow.com/questions/6196725/how-does-the-css-block-formatting-context-work)

[深入理解 CSS 外边距折叠（Margin Collapse）](https://tech.youzan.com/css-margin-collapse/)
