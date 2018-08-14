# XSS

XSS 全称跨站脚本(Cross Site Scripting), 为了不和层叠样式表(Cascading Style Sheets, CSS)的缩写混淆, 故缩写为 XSS.

跨站脚本攻击是一种常见的 web 安全漏洞, 它主要是指攻击者可以在页面中插入恶意脚本代码, 当受害者访问这些页面时, 浏览器会解析并执行这些恶意代码, 从而达到窃取用户身份/钓鱼/传播恶意代码等行为.

恶意内容一般包括 javascript, 但是, 有时候也会包括 html, flash. XSS 攻击的形式千差万别, 但是, 它们的共同点为: 将一些隐私数据像 cookie、session 发送给攻击者, 将受害者重定向到一个由攻击者控制的网站, 在受害者的机器上进行一些恶意操作.

## 分类

- 存储型(持久型)

  存储型 XSS 会把用户输入的数据 "存储" 在服务器端, 当浏览器请求数据时, 脚本从服务器上传回并执行.

  一个典型的例子是攻击者在留言板提交了包含恶意代码的留言, 当其他用户查看该留言时, 留言内容会从服务端返回, 并在浏览器解析执行恶意代码.

- 反射型(非持久型)

  反射型 XSS 是通过提交内容, 不经过数据库(与存储型相反), 直接反射回显在页面上.

  比如点击一个恶意链接, 跳转到一个攻击者预先准备好的页面, 该页面直接返回并执行一段恶意代码`<script>alert(反射型xss)</script>`

- 基于 DOM

  反射型的一种, 是指通过恶意脚本修改页面的 DOM 结构.

  比如有某个链接如下, 通过获取 url 中的 name 显示在页面中

  `http://site/index.html?name=xxx`

  ```javascript
  // 网站脚本
  window.onload = function() {
    var search = window.location.search;
    var index = search.indexOf("=");
    var name = search.substr(index + 1);
    document.write(decodeURIComponent(name));
  };
  ```

  把`name`修改为`<a href='http://www.google.com'>name</a>`, 当用户点击`name`时就会跳转到谷歌.

## 防范

- HttpOnly 防止劫取 cookie

  防止攻击者通过注入脚本获取用户 cookie 信息.

- 用户输入检查

  对用户输入进行检查, 过滤, 转义. 建立可信任的字符和 HTML 标签白名单, 对于不在白名单之列的字符或者标签进行过滤或编码.

  在 XSS 防御中, 输入检查一般是检查用户输入的数据中是否包含`<`,`>`等特殊字符, 如果存在, 则对特殊字符进行过滤或编码, 这种方式也称为 XSS Filter.

  在一些前端框架中, 都会有一份`decodingMap`, 用于对用户输入所包含的特殊字符或标签进行编码或过滤, 如 `<`, `>`, `script`, 防止 XSS 攻击：

  ```JavaScript
  // vuejs 中的 decodingMap
  // 在 vuejs 中,如果输入带 script 标签的内容,会直接过滤掉
  const decodingMap = {
  '&lt;': '<',
  '&gt;': '>',
  '&quot;': '"',
  '&amp;': '&',
  '&#10;': '\n'
  }
  ```

- 服务端输出检查

  一般来说, 除富文本的输出外, 在变量输出到 HTML 页面时, 可以使用编码或转义的方式来防御 XSS 攻击. 例如利用[sanitize-html](https://github.com/punkave/sanitize-html)对输出内容进行有规则的过滤之后再输出到页面中.

# CSRF

CSRF, 全称跨站请求伪造(Cross Site Request Forgery), 是一种劫持受信任用户向服务器发送非预期请求的攻击方式.

通常情况下, CSRF 攻击是攻击者借助受害者的 Cookie 骗取服务器的信任, 可以在受害者毫不知情的情况下以受害者名义伪造请求发送给受攻击服务器, 从而在并未授权的情况下执行在权限保护之下的操作.

## 原理

CSRF 攻击是黑客借助受害者的 cookie 骗取服务器的信任, 但是黑客并不能拿到 cookie, 也看不到 cookie 的内容. 另外, 对于服务器返回的结果, 由于浏览器同源策略的限制, 黑客也无法进行解析. 因此, 黑客无法从返回的结果中得到任何东西, 他所能做的就是给服务器发送请求, 以执行请求中所描述的命令, 在服务器端直接改变数据的值, 而非窃取服务器中的数据.

<img src="https://github.com/tzstone/MarkdownPhotos/blob/master/CSRF.png" align=center />

## 防范

CSRF 攻击之所以能够成功, 是因为黑客可以完全伪造用户的请求, 该请求中所有的用户验证信息都是存在于 cookie 中, 因此黑客可以在不知道这些验证信息的情况下直接利用用户自己的 cookie 来通过安全验证. 要抵御 CSRF, 关键在于在请求中放入黑客所不能伪造的信息, 并且该信息不存在于 cookie 之中.

- Referer Check

  根据 HTTP 协议, 在 HTTP 头中有一个字段叫`Referer`, 它记录了该 HTTP 请求的来源地址. 通过 Referer Check, 可以检查请求是否来自合法的"源"(假设黑客对某网站实施 CSRF 攻击, 他只能在自己的网站构造请求, 该请求的 Referer 是指向黑客自己的网站).

  `注意`:

  - Referer 的值是由浏览器提供的, 虽然 HTTP 协议上有明确的要求, 但是每个浏览器对于 Referer 的具体实现可能有差别, 并不能保证浏览器自身没有安全漏洞. 事实上, 对于某些浏览器, 比如 IE6 或 FF2, 目前已经有一些方法可以篡改 Referer 值.

  - 即便是使用最新的浏览器, 黑客无法篡改 Referer 值, 这种方法仍然有问题. 因为 Referer 值会记录下用户的访问来源, 有些用户认为这样会侵犯到他们自己的隐私权, 特别是有些组织担心 Referer 值会把组织内网中的某些信息泄露到外网中. 因此, 用户自己可以设置浏览器使其在发送请求时不再提供 Referer. 当他们正常访问银行网站时, 网站会因为请求没有 Referer 值而认为是 CSRF 攻击, 拒绝合法用户的访问.

- 添加 token 验证

  在 HTTP 请求中以参数的形式加入一个随机产生的 token,并在服务器端建立一个拦截器来验证这个 token, 如果请求中没有 token 或者 token 内容不正确, 则认为可能是 CSRF 攻击而拒绝该请求.

- 在 HTTP 头中自定义属性并验证

  这种方法也是使用 token 并进行验证, 和上一种方法不同的是, 这里并不是把 token 以参数的形式置于 HTTP 请求之中, 而是把它放到 HTTP 头中自定义的属性里. 通过 XMLHttpRequest 这个类, 可以一次性给所有该类请求加上 csrftoken 这个 HTTP 头属性, 并把 token 值放入其中. 这样解决了上种方法在请求中加入 token 的不便, 同时, 通过 XMLHttpRequest 请求的地址不会被记录到浏览器的地址栏, 也不用担心 token 会透过 Referer 泄露到其他网站中去.

## 参考资料

[浅说 XSS 和 CSRF](https://github.com/dwqs/blog/issues/68?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)

[跨站的艺术-XSS 入门与介绍](http://www.fooying.com/the-art-of-xss-1-introduction/)

[跨站脚本攻击(Cross-site scripting)](https://developer.mozilla.org/zh-CN/docs/Glossary/Cross-site_scripting)

[CSRF 攻击的应对之道](https://www.ibm.com/developerworks/cn/web/1102_niugang_csrf/index.html)
