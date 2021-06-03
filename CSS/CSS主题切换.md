# CSS主题切换

相信大家对网页的主题样式功能肯定不陌生。对于一些站点，在基础样式上，开发者还会为用户提供多种主题样式以供选择。

下面就是一个主题样式功能：用户可以在右侧选择自己喜欢的主题色，从而得到一个“个性”的页面。

![](https://user-gold-cdn.xitu.io/2018/12/12/1679e2f8c6f75eea?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果你也在产品中遇到了主题样式的相关需求，不妨先看看以下几点建议：

1. 尽可能避免这个功能。因为很多时候这可能只是个伪需求。
2. KISS原则（Keep It Simple, Stupid!）。尽可能降低其复杂性。
3. 尽量只去改变外观，而不要改动元素盒模型（box-model）。
4. 严格控制你的规则，避免预期外的差异。
5. 把它作为一个锦上添花的功能来向上促销（up-sell）。

## 实现主题样式的几种方式

### 方式一: Theme Layer

> Overriding default style with additional CSS.

这应该是实现主题功能的一种最常用的手段了。首先，我们的站点会有一个最初的`基础样式`（或者叫默认样式）；然后通过添加一些后续的额外的CSS来`覆盖`与重新定义部分样式。

#### 具体实现

首先，我们引入基础的样式 `components.*` 文件

```css
@import "components.tabs";
@import "components.buttons"
```

其中 `components.tabs` 文件内容如下:

```css
.tab {
    margin: 0;
    padding: 0;
    background-color: gray;
}
```

然后，假设我们的某个主题的样式文件存放于 `theme.*` 文件. 对应于 `components.tabs`，`theme.tabs` 文件内容如下:

```css
.tab {
    background-color: red;
}
```

因此，我们只需要引入主题样式文件即可:

```css
@import "components.tabs";
@import "components.buttons";

@import "theme.tabs";
```

这样当前的样式就变为了:

```css
.tab {
    margin: 0;
    padding: 0;
    /* background-color: gray; */
    background-color: red;
}
```

#### 优点

- 实现方式简单
- 可以实现将主题应用与所有元素

#### 缺点

- 过多的冗余代码
- 许多的CSS其实是无用的，浪费了带宽
- 把样式文件切分到许多文件中，更加琐碎

### 方式二：Stateful Theming

> Styling a UI based on a state or condition.

该方式可以实现基于条件选择不同的主题皮肤，并允许用户在客户端`随时切换主题`。非常适合需要客户端样式切换功能，或者需要对站点某一部分（区域）进行独立样式设置的场景。

#### 具体实现

还是类似上一节中 Tab 的这个例子，我们可以将 Tab 部分的 (S)CSS 改为如下形式：

```css
.tab {
    background-color: gray;

    .t-red & {
        background-color: red;
    }
    
    .t-blue & {
        background-color: blue;
    }
}
```

这里我们把`.t-red`与`.t-blue`称为 Tab 元素的`上下文环境（context）`。Tab 元素会根据 context 的不同展示出不同的样式。

最后我们给`body`元素加上这个开关:

```html
<body class="t-red">
    <ul class="tabs">...</ul>
</body>
```

此时 Tab 的颜色为红色。当我们将`t-red`改为`t-blue`时，Tab 就变为了蓝色主题。

进一步的，我们可以创建一些 (S)CSS 的 util class（工具类）来专门控制一些 CSS 属性，帮助我们更好地控制主题。例如我们使用如下的`.u-color-current`类来控制不同主题下的字体颜色:

```css
.u-color-current {
    .t-red & {
        color: red;
    }
    
    .t-blue & {
        color: blue;
    }
}
```

这样，当我们在不同主题上下文环境下使用`.u-color-current`时，就可以控制元素展示出不同主题的字体颜色:

```html
<body class="t-red">
    <h1 class="page-title u-color-current">...</h1>
</body>
```

上面这段代码会控制`<h1>`元素字体颜色为红色主题时的颜色。

#### 优点

- 将许多主题放在了同一处代码中
- 非常适合主题切换的功能
- 非常适合站点局部的主题化
- 可以实现将主题应用于所有元素

#### 缺点
- 有时有点也是缺点，将许多主题混杂在了同一块代码中
- 可能会存在冗余

### 方式三：Config Theming

> Invoking a theme based on settings.

这种方式其实是在开发侧来实现主题样式的区分与切换的。基于不同的配置，配合一些开发的自动化工具，我们可以在`开发时期`根据配置文件，`编译`生成不同主题的 CSS 文件。

它一般会结合使用一些 CSS 预处理器，可以对不同的 UI 元素进行主题分离，并且向客户端直接提供主题样式下最终的 CSS。

#### 具体实现

我们还是以 Sass 为例. 首先会有一份 Sass 的配置文件，例如`settings.config.scss`，在这份配置中定义当前的主题值以及一些其他变量

```css
$theme: red;
```

然后对于一个 Tab 组件，我们这么来写它的 Sass 文件:

```css
.tab {
    margin: 0;
    padding: 0;
    
    @if ($theme == red) {
        background-color: red;
    } @else {
        background-color: gray;
    }
}
```

这时，我们在其之前引入相应的配置文件后

```css
@import "settings.config";
@import "components.tabs";
```

Tab 组件就会呈现出红色主题。

当然，我们也可以把我们的`settings.config.scss`做的更健壮与易扩展一些:

```css
$config: (
    theme: red,
    env: dev,
)

// 从$config中获取相应的配置变量
@function config($key) {
    @return map-get($config, $key);
}
```

与之前相比，这时候使用起来只需要进行一些小的修改，将直接使用`theme`变量改为调用`config`方法

```css
.tab {
    margin: 0;
    padding: 0;
    
    @if (config(theme) == red) {
        background-color: red;
    } @else {
        background-color: gray;
    }
}
```

#### 优点

- 访问网站时，只会传输所需的 CSS，节省带宽
- 将主题的控制位置放在了一个地方（例如上例中的`settings.config.scss`文件）
- 可以实现将主题应用于所有元素

#### 缺点

- 在 Sass 中会有非常多逻辑代码
- 只支持有限数量的主题
- 主题相关的信息会遍布代码库中
- 添加一个新主题会非常费劲

### 方式四：Theme Palettes

> Holding entire themes in a palette file.

这种方式有些类似于我们绘图时，预设了一个`调色板（palette）`，然后使用的颜色都从其中取出一样。

在实现主题功能时，我们也会有一个类似的“调色板”，其中定义了主题所需要的各种属性值，之后再将这些信息注入到项目中。

当你经常需要为客户端提供完全的定制化主题，并且经常希望更新或添加主题时，这种模式会是一个不错的选择。

#### 具体实现

在方式三中，我们在一个独立的配置文件中设置了一些“环境”变量，来标示当前所处的主题。而在方式四中，我们会更进一步，抽取出一个专门的 palette 文件，用于存放不同主题的变量信息。

例如，现在我们有一个`settings.palette.red.scss`文件:

```css
$color: red;
$color-tabs-background: $color-red;
```

然后我们的`components.tabs.scss`文件内容如下

```css
.tabs {
    margin: 0;
    padding: 0;
    backgroung-color: $color-tabs-background;
}
```

这时候，我们只需要引入这两个文件即可

```css
@import "settings.palette.red";
@import "components.tabs";
```

可以看到，`components.tabs.scss`中并没有关于主题的逻辑判断，我们只需要专注于编辑样式，剩下就是选择所需的主题调色板（palette）即可。

#### 优点

- 编译出来的样式代码无冗余
- 非常适合做一些定制化主题，例如一个公司采购了你们的系统，你可以很方便实现一个该公司的主题
- 可以从一个文件中完全重制出你需要的主题样式

#### 缺点

- 由于主要通过设定不同变量，所以代码确定后，能实现的修改范围会是有限的

### 方式五：用户定制化 User Customisation

> Letting users style their own UIs.

这种模式一般会提供一个个性化配置与管理界面，让用户能自己定义页面的展示样式。

“用户定制化”在社交媒体产品、SaaS 平台或者是 Brandable Software 中最为常见。

#### 具体实现

要实现定制化，可以结合方式二中提到的 util class。

首先，页面中支持自定义的元素会被预先添加 `util class`，例如 Tab 元素中的`u-user-color-background`

```html
<ul class="tabs u-user-color-background">...</ul>
```

此时，`u-user-color-background`还并未定义任何样式。而当用户输入了一个背景色时，我们会创建一个`<style>`标签，并将 hex 值注入其中

```html
<style id="my-custom">
  .u-user-color-background {
      background-color: #00ffff;
  }
</style>
```

这时用户就得到了一个红色的 Tab。

Twitter 就是使用这种方式来实现用户定制化的界面样式的：

![](https://user-gold-cdn.xitu.io/2018/12/12/1679e300d854a05e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### 优点

- 不需要开发人员的输入信息（是用户定义的）
- 允许用户拥有自己“独一无二”的站点
- 非常实用

#### 缺点

- 不需要开发人员的输入信息也意味着你需要处理更多的“不可控”情况
- 会有许多的冗余
- 会浪费 CSS 的带宽
- 失去部分 CSS 的浏览器缓存能力

### 方式六：CSS自定义属性(变量)

CSS变量通常用于存储颜色，字体名称，字体大小，长度单位等。然后可以在样式表中的多个位置引用和重用它们。大多数开发人员都会引用"`CSS变量`"，但官方名称是"`自定义属性`"。

CSS自定义属性可以修改可在`整个样式表`中引用的变量。以前，只有使用Sass等CSS预处理器才能实现这一点。

`自定义属性`是一个名称以两个连字符（ - ）开头的属性，如 `--foo`。定义后可以使用 `var()` 引用变量。 让我们考虑这个例子：

```css
:root {
  --bg-color: #000;
  --text-color: #fff;
}
```

在`:root`选择器中定义自定义属性意味着它们可以作用于`全局文档中所有元素`。`:root`是一个CSS伪类，它匹配文档的根元素 – `<html>`元素。它类似于 html 选择器，但具有更高的优先级。

您可以在文档中的任何位置访问 `:root` 中的自定义属性的值：

```css
div {
  color: var(--text-color);
  background-color: var(--bg-color);
}
```

您还可以在CSS变量中包含回退值。例如：

```css
div {
  color: var(--text-color, #000);
  background-color: var(--bg-color, #fff);
}
```

如果未定义自定义属性，则使用其回退值代替。

除了 `:root` 或 `html` 选择器之外的CSS选择器内定义的自定义属性使变量可用于匹配元素及其子元素。

#### 具体实现

定义默认变量和主题变量:

```css
:root {
  --bg-color: #fff;
}

:root[theme='dark'] {
  --bg-color: #ccc
}

.bg {
  background-color: var(--bg-color);
}
```

html:

```html
<body>
  <button id="toggle-theme">toggle-theme</button>
  <p class="bg">test</p>
</body>
```

切换主题:

```javascript
const toggleBtn = document.querySelector("#toggle-theme");
toggleBtn.addEventListener('click', e => {
  if(document.documentElement.hasAttribute('theme')){
    document.documentElement.removeAttribute('theme');
  } else {
    document.documentElement.setAttribute('theme', 'dark');
  }
});
```

#### 优点

- 可以在单独位置定义值。
- 可以适当地命名该值以帮助维护和可读性。
- 可以使用JavaScript动态更改该值。

#### 缺点

- 兼容性较差, 不支持ie

### element ui的动态换肤

#### 具体实现

生成一套主题， 将主题配色配置写在js中，在浏览器中用js动态修改style标签覆盖原有的CSS。

1. 准备一套默认theme.css样式

```css
/* theme.css */
.title {
  color: #FF0000
}
```

2. 准备主题色配置
```javascript
var colors = {
    red: {
        themeColor: '#FF0000'
    },
    blue: {
        themeColor: '#0000FF'
    }
}
```

3. 异步获取 theme.css ，将颜色值替换为关键词. 关键字可以确保以后能多次换色

```javascript
var styles = ''
axios.get('theme.css').then((resp=> {
 const colorMap = {
   '#FF0000': 'themeColor'
 }
 styles = resp.data
 Object.keys(colorMap).forEach(key => {
   const value = colorMap[key]
   styles = styles.replace(new RegExp(key, 'ig'), value)
 })
}))
```

style 变为：

```css
.title {
  color: theme-color
}
```

4. 把关键词再换回刚刚生成的相应的颜色值，并在页面上添加 style 标签

```javascript
 // console 中执行 writeNewStyle (styles, colors.blue)  即可变色
 function writeNewStyle (originalStyle, colors) {
    let oldEl = document.getElementById('temp-style')
    let cssText = originalStyle
    // 替换颜色值
    Object.keys(colors).forEach(key => {
    cssText = cssText.replace(new RegExp(key, 'ig'), colors[key])
    })
    const style = document.createElement('style')
    style.innerText = cssText
    style.id = 'temp-style'

    oldEl ? document.head.replaceChild(style, oldEl) : 
    document.head.appendChild(style)  // 将style写入页面
}
```

此时style 变为：

```css
.title {
  color: '#0000FF'
}
```

#### 优点

- 只需一套CSS文件
- 换肤不需要延迟等候
- 可自动适配多种主题色

#### 缺点

- 稍难理解
- 需准确的css颜色值
- 可能受限于浏览器性能

### 主题样式切换

#### 替换css

先将默认主题写到页面中, 通过js触发事件加载其他主题的css。后加载的css会覆盖之前样式从而达到切换主题的效果。

举个栗子, 默认主题是亮色主题:

```html
<link rel="stylesheet" type="text/css" href="/css/nutzbs_light.css">
```

实现添加/删除暗色主题:

```javascript
// 添加暗色主题
function addDarkTheme() {
   var link = document.createElement('link');
   link.type = 'text/css';
   link.id = "theme-css-dark";  // 加上id方便后面好查找到进行删除
   link.rel = 'stylesheet';
   link.href = '/css/nutzbs_dark.css';
   document.getElementsByTagName("head")[0].appendChild(link);
}
// 删除暗色主题
function removeDarkTheme() {
   $('#theme-css-dark').remove();
}
```

切换主题:

```javascript
// 使用暗色主题
function useDarkTheme(useDark) {
   if (useDark) {
       addDarkTheme();
   } else {
       removeDarkTheme();
   }
}
```

直接通过加载css切换主题后可能会面临一个问题: 切换到其他页面后又变成了默认主题，所以需要`保持`当前选中的主题. 可以采用`localStorage`, `cookie`等方式进行记录. 下面采用cookie的方式:

```javascript
// 获取cookie中选中的主题名称，没有就给个默认的
function getThemeCSSName() {
   return Cookies.get('nutzam-theme') || "light";
}

// 使用暗色主题(记录选择到cookie中)
function useDarkTheme(useDark) {
   Cookies.set('nutzam-theme', useDark ? "dark" : "light");
   if (useDark) {
       addDarkTheme();
   } else {
       removeDarkTheme();
   }
}
```

然后在页面加载完成后，调用一下该方法即可, 切换其他页面也就自动的切换到选中的主题了:

```javascript
$(document).ready(function () {
    useDarkTheme(getThemeCSSName() == 'dark');
})
```

#### 插入style

如方式五

#### 修改var变量

如方式六

### 方案选择

针对以上六种实现方式，在面对产品需求，我们应该如何选择呢？

这里有一个不是非常严谨的方式可以参考。你可以通过尝试问自己下面这几个问题来做出决定：

- 是你还是用户谁来确定样式？用户：选择【方式五】User Customisation 或【方式六】CSS变量

- 主题是否会在客户端中被切换？是：选择【方式二】Stateful Theming 或【方式五】User Customisation 或【方式六】CSS变量

- 是否有主题能让用户切换？是：选择【方式二】Stateful Theming或【方式六】CSS变量

- 你是希望网站的某些部分需要有不同么？是：选择【方式二】Stateful Theming

- 是否有预设的主题让客户端来选择？是：选择【方式三】Config Theming 或【方式六】CSS变量

- 是否是类似“贴牌”这类场景？是：选择【方式一】Theme Layer 或【方式四】Theme Palettes

参考资料:

[(S)CSS中实现主题样式的4½种方式 [译]](https://juejin.cn/post/6844903735605329934)

[web网页中主题切换的实现思路](https://www.jianshu.com/p/fe807f5ef394)

[CSS自定义属性怎样实现主题切换？](https://zhuanlan.zhihu.com/p/60975003)

[如何简单实现 CSS 主题的切换](https://blog.p2hp.com/archives/7293)

[一文总结前端换肤换主题](https://www.jianshu.com/p/35e0581629d2)