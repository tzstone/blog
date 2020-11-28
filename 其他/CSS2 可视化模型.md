# CSS2 可视化模型

## 画布(canvas)

根据 [CSS2 规范](http://www.w3.org/TR/CSS21/intro.html#processing-model)，“画布”这一术语是指“用来呈现格式化结构的空间”，也就是供浏览器绘制内容的区域。画布的空间尺寸大小是无限的，但是浏览器会根据视口的尺寸选择一个初始宽度。

根据 www.w3.org/TR/CSS2/zindex.html ，画布如果包含在其他画布内，就是透明的；否则会由浏览器指定一种颜色。

## CSS 盒模型

[CSS 盒模型](http://www.w3.org/TR/CSS2/box.html)描述的是针对文档树中的元素而生成，并根据可视化格式模型进行布局的矩形框。
每个框都有一个内容区域（例如文本、图片等），还有可选的周围补白、边框和边距区域。

![CSS2 盒模型](https://github.com/tzstone/MarkdownPhotos/raw/master/CSS2%20%E6%A1%86%E6%A8%A1%E5%9E%8B.jpg)

每一个节点都会生成 0..n 个这样的框。
所有元素都有一个“display”属性，决定了它们所对应生成的框类型。示例：

```code
block - generates a block box.
inline - generates one or more inline boxes.
none - no box is generated.
```

默认值是 inline，但是浏览器样式表设置了其他默认值。例如，“div”元素的 display 属性默认值是 block。您可以在这里找到默认样式表示例：www.w3.org/TR/CSS2/sample.html

框的布局方式是由以下因素决定的：

- 框类型
- 框尺寸
- 定位方案
- 外部信息，例如图片大小和屏幕大小

### 框类型

- block 框：形成一个 block，在浏览器窗口中拥有其自己的矩形区域。

  ![block 框](https://github.com/tzstone/MarkdownPhotos/raw/master/block%20%E6%A1%86.png)

- inline 框：没有自己的 block，但是位于容器 block 内。

  ![inline 框](https://github.com/tzstone/MarkdownPhotos/raw/master/inline%20%E6%A1%86.png)

block 采用的是一个接一个的垂直格式，而 inline 采用的是水平格式。

![block 和 inline 格式](https://github.com/tzstone/MarkdownPhotos/raw/master/block%20%E5%92%8C%20inline%20%E6%A0%BC%E5%BC%8F.png)

inline 框放置在行中或“行框(line boxes)”中。这些行至少和最高的框一样高，还可以更高，当框根据“底线”对齐时，这意味着元素的底部需要根据其他框中非底部的位置对齐。如果容器的宽度不够，inline 元素就会分为多行放置。在段落中经常发生这种情况。

![行](https://github.com/tzstone/MarkdownPhotos/raw/master/%E8%A1%8C.png)

## 定位方案

有三种定位方案：

1. 普通：根据对象在文档中的位置进行定位，也就是说对象在渲染树中的位置和它在 DOM 树中的位置相似，并根据其框类型和尺寸进行布局。
2. 浮动：对象先按照普通流进行布局，然后尽可能地向左或向右移动。
3. 绝对：对象在渲染树中的位置和它在 DOM 树中的位置不同。

定位方案是由“position”属性和“float”属性设置的。

- 如果值是 static 和 relative，就是普通流
- 如果值是 absolute 和 fixed，就是绝对定位

static 定位无需定义位置，而是使用默认定位。对于其他方案，网页作者需要指定位置：top、bottom、left、right。

### 相对定位

先按照普通方式定位，然后根据所需偏移量进行移动。

![相对定位](https://github.com/tzstone/MarkdownPhotos/raw/master/%E7%9B%B8%E5%AF%B9%E5%AE%9A%E4%BD%8D.png)

### 浮动

浮动框会移动到行的左边或右边。有趣的特征在于，其他框会浮动在它的周围。下面这段 HTML 代码：

```html
<p>
  <img style="float:right" src="images/image.gif" width="100" height="100" />
  Lorem ipsum dolor sit amet, consectetuer...
</p>
```

显示效果如下：
![浮动](https://github.com/tzstone/MarkdownPhotos/raw/master/%E6%B5%AE%E5%8A%A8.png)

### 绝对定位和固定定位

这种布局是准确定义的，与普通流无关。元素不参与普通流。尺寸是相对于容器而言的。在固定定位中，容器就是可视区域。

![固定定位](https://github.com/tzstone/MarkdownPhotos/raw/master/%E5%9B%BA%E5%AE%9A%E5%AE%9A%E4%BD%8D.png)

请注意，即使在文档滚动时，固定框(fixed box)也不会移动。

## 分层展示

这是由 z-index CSS 属性指定的。它代表了框的第三个维度，也就是沿“z 轴”方向的位置。

这些框分散到多个堆栈（称为堆栈上下文）中。在每一个堆栈中，会首先绘制后面的元素，然后在顶部绘制前面的元素，以便更靠近用户。如果出现重叠，新绘制的元素就会覆盖之前的元素。
堆栈是按照 z-index 属性进行排序的。具有“z-index”属性的框形成了本地堆栈。视口具有外部堆栈。

示例：

```html
<style type="text/css">
      div {
        position: absolute;
        left: 2in;
        top: 2in;
      }
</style>

<p>
    <div
         style="z-index: 3;background-color:red; width: 1in; height: 1in; ">
    </div>
    <div
         style="z-index: 1;background-color:green;width: 2in; height: 2in;">
    </div>
 </p>
```

结果如下：

![固定定位](https://github.com/tzstone/MarkdownPhotos/raw/master/%E5%9B%BA%E5%AE%9A%E5%AE%9A%E4%BD%8D1.png)

虽然红色 div 在标记中的位置比绿色 div 靠前（按理应该在常规流程中优先绘制），但是 z-index 属性的优先级更高，因此它移动到了根框所保持的堆栈中更靠前的位置。

参考资料

[How Browsers Work: Behind the Scenes of Modern Web Browsers](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/)
