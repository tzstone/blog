# [译]Resource Timing 实用技巧

自 2012 年,[W3C Web 性能工作小组](https://www.w3.org/webperf/)为我们带来了[Navigation Timing](https://www.w3.org/TR/navigation-timing/)，现在几乎所有主流浏览器都可以使用它。 `Navigation Timing` 定义了一个 JavaScript API，用于测量主页面的性能。例如：

```js
// Navigation Timing
var t = performance.timing,
  pageloadtime = t.loadEventStart - t.navigationStart,
  dns = t.domainLookupEnd - t.domainLookupStart,
  tcp = t.connectEnd - t.connectStart,
  ttfb = t.responseStart - t.navigationStart;
```

拥有主页面的计时指标很棒，但要诊断现实世界的性能问题，通常需要深入研究单个资源。因此，我们有更新的[Resource Timing 规范](http://www.w3.org/TR/resource-timing/)。该 JavaScript API 提供与`Navigation Timing`类似的计时信息，但它是为每个单独的资源提供的。举个栗子：

```js
// Resource Timing
var r0 = performance.getEntriesByType("resource")[0],
  loadtime = r0.duration,
  dns = r0.domainLookupEnd - r0.domainLookupStart,
  tcp = r0.connectEnd - r0.connectStart,
  ttfb = r0.responseStart - r0.startTime;
```

截至今天(译者注: 原文写于 2014 年 8 月)，`Chrome`，`Chrome for Android`，`Opera`，`IE10`和`IE11`都支持`Resource Timing`。这可能超过您流量的 50％，因此应该能够提供足够的数据来发现您网站中低性能的资源。

使用`Resource Timing`似乎很简单，但是当我编写第一个生产质量的`Resource Timing`代码时，我遇到了几个我要分享的问题。以下是我在现实世界中跟踪`Resource Timing`指标的实用技巧。

### 1. 使用`getEntriesByType("resource")`而不是`getEntries()`

您可以通过获取当前页面的资源计时性能对象集来开始使用资源计时。许多资源计时示例使用`performance.getEntries()`来执行此操作，暗示着此调用仅返回资源计时对象。但实际上`getEntries()`可能会返回四种类型的计时对象：`resource`, `navigation`, `mark`, 和`measure`。

到目前为止，这并没有给开发人员带来问题，因为`resource`是大多数页面中唯一的条目类型。 `navigation`条目类型是[Navigation Timing 2](http://www.w3.org/TR/navigation-timing-2/)的一部分，据我所知未在任何浏览器中实现。 `mark`和`measure`条目类型来自[User Timing](http://www.w3.org/TR/user-timing/)规范，该规范在某些浏览器中可用, 但未广泛使用。(译者注: 原文写于 2014.8, 以上情况可能与现在(2018.8)不一致)

换句话说，`getEntriesByType("resource")`和`getEntries()`今天可能会返回相同的结果。但是`getEntries()`很可能会很快返回混合的性能对象类型。最好使用`performance.getEntriesByType("resource")`，这样您就可以确保只返回资源计时对象。

### 2. 使用`Navigation Timing`来测量主页面的请求

在获取网页时，通常会有对主 HTML 文档的请求。但是，`performance.getEntriesByType("resource")`并不返回该资源。要获取有关主 HTML 文档的计时信息，您需要使用 `Navigation Timing` 对象（`performance.timing`）。

虽然不太可能，但如果没有条目，这可能会导致错误。例如，我之前的`Resource Timing`示例使用了以下代码：

```js
performance.getEntriesByType("resource")[0];
```

如果页面的唯一资源是主 HTML 文档，那么`getEntriesByType("resource")`将返回一个空数组，引用元素[0]会导致 JavaScript 错误。如果您没有零资源的页面，那么您可以在[http://fast.stevesouders.com/](http://fast.stevesouders.com/)上进行测试。

### 3. 注意`secureConnectionStart`的问题

[secureConnectionStart](http://www.w3.org/TR/resource-timing/#dom-performanceresourcetiming-secureconnectionstart)属性允许我们测量 SSL 协商所需的时间。这很重要--我经常看到 SSL 协商时间为 500 毫秒或更长。 `secureConnectionStart` 有三个可能的值：

- 如果此属性不可用，则必须将其设置为`undefined`
- 如果未使用 HTTPS，则必须将其设置为`0`
- 如果此属性可用且使用了 HTTPS，则必须将其设置为时间戳

关于`secureConnectionStart`, 有三件事需要了解。

第一，在 IE 浏览器中，`secureConnectionStart`的值始终是`undefined`，因为它不可用（该值被隐藏在 [WinINet](<http://msdn.microsoft.com/en-us/library/windows/desktop/aa383630(v=vs.85).aspx>) 中）。

第二，Chrome 中存在一个 bug，会错误地将 `secureConnectionStart` 设置为零。如果使用预先存在的 HTTPS 连接获取资源，`secureConnectionStart` 会被设置为零, 而实际上应该是时间戳。 （有关详细信息，请参阅[bug 404501](https://code.google.com/p/chromium/issues/detail?id=404501).）为避免数据不准确，请在测量 SSL 协商时间之前确保 `secureConnectionStart` 不是 `undefined` 也不是 `0`：

```js
var r0 = performance.getEntriesByType("resource")[0];
if (r0.secureConnectionStart) {
  var ssl = r0.connectEnd - r0.secureConnectionStart;
}
```

第三，关于[这一行](http://www.w3.org/TR/resource-timing/#dom-performanceresourcetiming-secureconnectionstart)，规范有误导性："...如果当前页面(`current page`)的方案是 HTTPS，则该属性必须在用户代理启动握手过程之前立即返回时间..."。当前页面可能是 HTTP 并且仍然包含我们要测量 SSL 协商时间的 HTTPS 资源。因此, 规范应更改为"...如果资源(`resource`)的方案是 HTTPS，则该属性必须在用户代理启动握手过程之前立即返回时间..."。幸运的是，浏览器的行为符合更正后的规范，换句话说，即使在 HTTP 页面上，`secureConnectionStart` 也可用于 HTTPS 资源。

### 4. 对跨域资源添加`Timing-Allow-Origin`

出于隐私原因，获取`Resource Timing`详细信息时存在[跨域限制](http://www.w3.org/TR/resource-timing/#cross-origin-resources)。默认情况下，资源的域与主页面的域不同时，以下属性会被设置为 0：

- redirectStart
- redirectEnd
- domainLookupStart
- domainLookupEnd
- connectStart
- connectEnd
- secureConnectionStart
- requestStart
- responseStart

在某些情况下，需要测量跨域资源的性能，例如，当网站为其 CDN 使用不同的域（例如“youtube.com”使用“s.ytimg.com”）以及使用了第三方资源（例如“ajax.googleapis.com”）。如果资源返回 `Timing-Allow-Origin` 响应头，则允许访问跨域资源的详细计时信息。此响应头指定允许查看详细计时信息的主页面域名列表，但在大多数情况下，通配符（“`*`”）用于授予对所有源的访问权限。例如，http：//ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min.js 的 `Timing-Allow-Origin` 响应头是：

```js
Timing-Allow-Origin: *
```

当第三方资源添加此响应头时，它允许网站所有者测量和了解在其网页上引入第三方资源的性能。正如 Ilya Grigorik 报道的那样，有几个第三方添加了这个响应头。以下是一些指定`Timing-Allow-Origin: *`的示例资源：

- Google Hosted Libraries: http://ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min.js
- Google+ widgets: https://apis.google.com/js/plusone.js
- Google Fonts: http://fonts.gstatic.com/s/opensans/v9/DXI[snip…]N3Vs.woff2
- Facebook widgets: http://connect.facebook.net/en_US/all.js
- Disqus widgets: http://go.disqus.com/embed.js

在测量`Resource Timing`时，确定是否可以访问受限制的计时属性非常重要。您可以通过测试任何受限制的属性（上文所列出的, 除了`secureConnectionStart` 以外）是否为 0 来实现。我总是使用 `requestStart`。这是早期代码的更好版本，它在计算更详细的性能指标之前检查受限属性是否可用：

```js
// Resource Timing
var r0 = performance.getEntriesByType("resource")[0],
  loadtime = r0.duration;
if (r0.requestStart) {
  var dns = r0.domainLookupEnd - r0.domainLookupStart,
    tcp = r0.connectEnd - r0.connectStart,
    ttfb = r0.responseStart - r0.startTime;
}
if (r0.secureConnectionStart) {
  var ssl = r0.connectEnd - r0.secureConnectionStart;
}
```

这些检查非常重要。否则，如果假设您可以访问这些受限制的属性，则实际上不会出现任何错误;你只会得到虚假的数据。由于资源被限制访问时, 其值会被设置为零，因此`domainLookupEnd - domainLookupStart`之类的内容会转换为`0 - 0`，这会返回一个似是而非的结果（`0`），这不一定是的真实 DNS 查询时间（在这种情况下）。这会导致您的指标具有过多的`0`值，这会错误地让聚合统计数据看起来比实际更加乐观。

### 5. 理解 `0` 意味着什么

如＃4 中所述，对于访问受限的跨域资源，某些`Resource Timing`属性会被设置为 0。同样，在查看详细属性之前检查该状态非常重要。但即使是可以访问的受限属性，您计算的指标也可能会是 0 值，了解这意味着什么很重要。

例如，（假设没有访问限制）`domainLookupStart` 和 `domainLookupEnd` 的值是时间戳。这两个值的差值是此资源执行 DNS 解析所花费的时间。通常，页面中只有一个资源对给定 hostname 的 DNS 解析时间是非 0 值。这是因为第一次 DNS 解析后浏览器会将解析结果缓存起来; 后续所有请求都使用该缓存。由于 DNS 解析的缓存是跨页面的，因此, 对于某个页面, 可能整个页面所有资源的 DNS 解析时间都是 0。`结论：D​​NS 解析时间是 0 意味着它是从缓存中读取的。`

类似地，对于给定的 hostname, 如果是重新使用了预先存在的 TCP 连接, 那么建立 TCP 连接（`connectEnd - connectStart`）的时间将是 0。每个 hostname 可能有 `0~6` (译者注:  同一域名下的最大并发请求数)个唯一的 TCP 连接，这些连接对应了 `0~6` 个非 0 的 TCP 连接计时时间，但由于该 hostname 上后续的所有请求将使用预先存在的连接，因此可能会有某个 TCP 连接时间为 0。`结论：TCP 连接时间为 0 意味着重新使用了预先存在的 TCP 连接。`

这同样适用于计算 SSL 协商时间（`connectEnd - secureConnectionStart`）。对于同一个 hostname, 最多有 6 个资源的 SSL 协商时间不为 0，但所有后续资源的 SSL 协商时间可能为 0，因为它们使用了预先存在的 HTTPS 连接。

最后，如果 `duration` 属性为零，则可能意味着资源是从缓存中读取的。

### 6. 确定是否正在测量 304 响应

Chrome 稳定版（版本 36）中的另一个 bug 已在第 37 版中得到修复。此问题即将消失，但由于大多数用户都在 Chrome 稳定版上，因此您当前的指标可能与用户的实际情况不同。这个 bug 是：一个跨域资源的 200 响应上带有 `Timing-Allow-Origin`, 但在其 304 响应中, 该响应头不会被考虑进去。因此，在 304 响应中, 所有受限的属性都被设置为 0，[测试页面在此](http://stevesouders.com/tests/tao.php)。

这不应该发生，因为缓存的 200 响应中的 `Timing-Allow-Origin` 头也应该（由浏览器）应用于 304 响应。这是 IE 中发生的情况。 （请尝试在 IE 10 或 11 中进行确认, [测试页面在此](http://stevesouders.com/tests/tao.php)。）

这会影响您的 Chrome 资源计时结果，如下所示：

- 如果您正在检查受限属性是否为 0（如＃4 中所述），那么您将跳过测量这些 304 响应。这意味着你只测量了 200 响应。但是 200 响应比 304 响应慢，因此您的`Resource Timing`合计测量结果将比实际更大。
- 如果您没有检查受限属性是否为 0，那么您将记录许多零值，这些值比实际上的 304 响应更快，因此您的`Resource Timing`统计结果将过于乐观。

没有简单的方法来避免这些偏差，好消息是这个 bug 已经修复了。您可能会尝试在 304 响应中发送 `Timing-Allow-Origin`。不幸的是，流行的 Apache Web 服务器无法在 304 响应中发送此响应头（请参阅[bug 51223](https://issues.apache.org/bugzilla/show_bug.cgi?id=51223)）。通过查看＃4 中列出的第三方资源，可以找到 304 响应中缺少 `Timing-Allow-Origin` 的进一步证据。如上所述，这些第三方资源在 200 响应中返回该响应头是很棒的，但是这 5 个资源中有 4 个没有在 304 响应中返回 `Timing-Allow-Origin`。在 Chrome 37 变得稳定之前，由于受限属性的缺失细节，`Resource Timing`指标可能会偏高或过低。幸运的是，`duration` 的值无论如何都是准确的。

### 7. 看看 `Boomerang`

如果您正在考虑编写自己的`Resource Timing`代码，首先应该查看 [Boomerang 的 Resource Timing 插件](http://www.lognormal.com/boomerang/doc/api/restiming.html)。 （代码在 [GitHub](https://github.com/lognormal/boomerang/blob/master/plugins/restiming.js) 上。）`Boomerang` 是 `Philip Tellis` 维护的一个流行的开源 RUM 包。他最初在 Yahoo 的时候就将其开源了. 但是现在正在提供持续的维护和增强，作为他在 SOASTA 的商业版（[mPulse](http://www.soasta.com/products/mpulse/)）的一部分。其代码清晰，简洁，强大，并解决了上述许多问题。

总之，`Navigation Timing`和`Resource Timing`是一些出色的新规范，可以让网站所有者更加了解其网页的性能。`Resource Timing`是这两个规范中较新的一个，因此仍然有一些瑕疵需要解决。这些提示将帮助您从`Resource Timing`指标中获得最大收益。我们建议您立即开始跟踪这些指标，以了解您的网站对最重要的人的表现：真实用户。

**更新**：以下是一些其他第三方资源，包含有`Timing-Allow-Origin` 响应头，从而允许网站所有者衡量他们对此第三方资源的表现：

- Boomerang: http://c.go-mpulse.net/boomerang/CEN[snip…]YQE
- Typekit: https://use.typekit.net/previewkits/pk-v1.js

**更新**：请注意，`getEntriesByType("resource")`和 `getEntries()`不包含 iframe 内的资源。如果 iframe 来自同一个域名，那么父窗口可以使用 iframe 的 `contentWindow.performance` 对象来访问它们。

#### 英文原文: [Resource Timing practical tips](https://www.stevesouders.com/blog/2014/08/21/resource-timing-practical-tips/)
