# Service Worker

Service Worker 是您的浏览器在后台运行一个脚本，，与网页分开，为不需要网页或用户交互的功能打开了大门。今天，它们已经包含了诸如推送通知和后台同步之类的功能。将来，Service Worker 可能会支持其他功能，例如定期同步或地理围栏。本教程讨论的核心功能是拦截和处理网络请求的能力，包括以编程方式管理响应缓存。之所以如此令人兴奋，是因为它允许您支持脱机体验，从而使开发人员可以完全控制体验。

在 service worker 出现之前，还有一个名为 [AppCache](https://www.html5rocks.com/en/tutorials/appcache/beginner/) 的 API 可以为用户提供离线 web 体验。AppCache API 存在许多问题，而 service worker 的设计是为了避免这些问题。

有关 service worker 的注意事项：

- 它是 [JavaScript Worker](https://www.html5rocks.com/en/tutorials/workers/basics/)，因此无法直接访问 DOM。然而，service worker 可以响应由 postMessage 接口发送过来的消息, 通过这样的方式与其控制的页面进行通信, 如果需要，这些页面可以操纵 DOM。

- Service Worker 是可编程的网络代理，使您可以控制如何处理页面中的网络请求。

- 它在不使用时会终止，在下一次需要时会重新启动，因此您不能依赖 Service Worker 的 onfetch 和 onmessage 处理程序中的全局状态。 如果您需要保留信息并在重新启动之间重复使用，则 service worker 可以访问 IndexedDB API。

先决条件:

- 浏览器支持

  Chrome，Firefox 和 Opera 支持 service worker。您可以在 Jake Archibald 的 [Serviceworker 是否已就绪](https://jakearchibald.github.io/isserviceworkerready/) 网站上跟踪所有浏览器的进度。

- 您需要 HTTPS

  在开发过程中，您将可以通过 localhost 使用 Service Worker，但是要将其部署在站点上，则需要在服务器上设置 HTTPS。

  使用 Service Worker，您可以劫持连接，构造和过滤响应。功能强大的东西。尽管您会充分利用这些能力，但中间人可能不会。为避免这种情况，您只能在通过 HTTPS 服务的页面上注册 Service Worker，因此我们知道浏览器在通过网络的过程中并未被篡改。

Service Worker 具有缓存 API, 给开发者提供了直接操作缓存的方式. 仅当页面安装了 Service Worker, Service Worker Cache 才会存在. Service Worker Cache 与 Memory Cache 的不同在于 Service Worker Cache 是`永久性`的, 即使关闭 tab 或者重启浏览器, 缓存仍然会保留.

有两种情况会`清除` Service Worker Cache:

- 开发者手动调用 API cache.delete(resource)
- 浏览器耗尽存储空间, 在这种情况下, 整个 Service Worker Cache 和所有其他原始存储(比如 indexedDB, localStorage 等)都会被清除.

## 作用域(scope)

Service Worker 负责特定作用域(scope)，该作用域最多限于单个主机(single host)。 因此，Service Worker 只能响应该作用域内的文档所发出的请求。

service worker 注册的默认作用域(scope)是其脚本(script) URL 的相对路径 `./`。这意味着，如果您在 `//example.com/foo/bar.js` 注册 Service Worker，则其默认作用域为 `//example.com/foo/`。

我们称页面(pages)，workers 和 shared workers 为 `客户端(clients)`。 您的 service worker 只能控制作用域内(in-scope)的 clients。 一旦 clients 被“控制”，它的 fetches (页面和子资源)将通过作用域内的 service worker 进行。您可以通过检测 `navigator.serviceWorker.controller` 来判断一个 clients 是否被控制，该方法将返回 null 或者一个 service worker 实例。

## 生命周期(lifecycle)

service worker 的生命周期与您的网页完全独立。

要为您的网站安装 service worker，您需要在页面的 JavaScript 中对其进行注册。注册 service worker 将导致浏览器在后台启动 service worker 安装步骤。

通常，在安装步骤中，您需要缓存一些静态资产。如果所有文件都已成功缓存，则 Service Worker 将变为 installed。如果任何文件无法下载和缓存，则安装步骤将失败并且 Service Worker 将不会激活（即不会安装）。如果发生这种情况，请放心，它将会在下次重试。但这意味着，如果安装了，您就知道缓存中有这些静态资产了。

安装后(installed)，将执行激活步骤，这是处理旧缓存的绝好机会。

激活步骤之后，Service Worker 将控制属于其作用域(scope)内的所有页面，但是首次注册该 Service Worker 的页面在再次加载之前不会受到控制。一旦 Service Worker 处于控制状态，它将处于以下两种状态之一：Service Worker 将被终止以节省内存，或者将处理从页面发出网络请求或消息时发生的 fetch 和 message 事件。

以下是首次安装时 Service Worker 生命周期的过度简化版本:

![lifecycle](https://developers.google.com/web/fundamentals/primers/service-workers/images/sw-lifecycle.png)

### 目的

- 使离线优先成为可能
- 允许一个新的 service worker 在不中断当前 service worker 的情况下做好准备
- 确保作用域(scope)内的页面始终由同一个 service worker（或没有 service worker）控制
- 确保站点一次只运行一个版本(避免先后打开多个 tab, 不同 tab 中站点版本不一致)

### 过程

- 注册, 下载, 解析, 执行

  当您调用 `.register()` 时，您的第一个 service worker 将会下载。如果您的脚本下载，解析失败或者在初始执行中抛出异常, 则注册 promise 将被 reject，并且 service worker 将被丢弃。

  ```code
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', function() {
      navigator.serviceWorker.register('/sw.js').then(function(registration) {
        // Registration was successful
        console.log('ServiceWorker registration successful with scope: ', registration.scope);
      }, function(err) {
        // registration failed :(
        console.log('ServiceWorker registration failed: ', err);
      });
    });
  }
  ```

  此代码检查 service worker API 是否可用，如果可用，则在页面加载后 `/sw.js` 的 service worker 将被注册。

  您可以在每次加载页面时调用 register（），而无需担心；浏览器将确定 service worker 是否已注册，并进行相应处理。

  register（）方法的一个微妙之处是 service worker 的位置。在这个例子中，您会注意到 service worker 文件位于域(domain)的根目录。 这意味着 service worker 的作用域(scope)将是整个来源(origin)。`换句话说，此 service worker 将收到此域上所有内容的 fetch 事件。`如果我们在 `/example/sw.js` 上注册 service worker 文件，则 service worker 只会看到 URL 以 `/example/` 开头（即 `/example/page1/`，`/example/page2/`）的页面的 fetch 事件。

  现在，您可以转到 chrome://inspect/＃service-workers 并查找您的站点，以检查是否启用了 service worker。

  您可能会发现在隐身窗口中测试 service worker 很有用，这样您就可以关闭并重新打开，而无需担心先前的 service worker 不会影响新窗口。 关闭隐身窗口后，从隐身窗口中创建的所有注册和缓存都将被清除。

- 安装(install)

  service worker 获得的第一个事件是 `install`。worker 执行时会立即触发该事件，并且每个 service worker `只调用一次`。 如果`更改` service worker 脚本，浏览器会认为它是`另一个` service worker，并且它将获得自己的 install 事件。

  install 事件是您在能够控制客户端之前缓存所有需要的内容的机会。

  在下面代码，您可以看到我们用所需的缓存名称调用 caches.open() ，然后调用 cache.addAll() 并传入文件数组。这是一个 promise 链（caches.open() 和 cache.addAll()）。`event.waitUntil()` 方法接受一个 promise，并使用它来知道安装需要多长时间以及安装是否成功。

  如果所有文件均已成功缓存，则将安装 service worker。如果任何文件下载失败(promise 被 reject)，则安装步骤将失败。这使您可以依靠拥有所有定义的资源(assets)，但是这确实意味着您需要注意在安装步骤中决定要缓存的文件列表。定义一长串文件将增加一个文件可能无法缓存的可能性，从而导致 service worker 无法安装。

  在 install 回调中，我们需要执行以下步骤：

  - 打开一个缓存。
  - 缓存我们的文件。
  - 确认是否已缓存所有必需的资源(assets)。

  ```code
  var CACHE_NAME = 'my-site-cache-v1';
  var urlsToCache = [
    '/',
    '/styles/main.css',
    '/script/main.js'
  ];

  self.addEventListener('install', function(event) {
    // Perform install steps
    event.waitUntil(
      caches.open(CACHE_NAME)
        .then(function(cache) {
          console.log('Opened cache');
          return cache.addAll(urlsToCache);
        })
    );
  });
  ```

- 激活(activate)

  一旦您的 service worker 准备控制客户端并处理诸如 `push` 和 `sync` 的事件，您将获得一个激活(`activate`)事件(active 之后才会接收 fetch 和 push 之类的事件)。但这并不意味着调用 `.register()` 的页面将受到控制(页面第一次加载时不受控制)。

  第一次加载 [demo](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/) 时，即使在 service worker 激活很长时间之后才请求 dog.svg，它也无法处理该请求，并且您仍然可以看到该狗的图像。默认值是`一致性(consistency)`，如果您的页面加载时没有 service worker，那么页面的子资源也不会通过 service worker。如果您第二次加载 demo（换句话说，刷新页面），它将受到控制。页面和图像都会经历 fetch 事件，您会看到一只猫。

  ```code
  // demo code

  self.addEventListener('install', event => {
  console.log('V1 installing…');

  // cache a cat SVG
  event.waitUntil(
    caches.open('static-v1').then(cache => cache.add('/cat.svg'))
  );
  });

  self.addEventListener('activate', event => {
  console.log('V1 now ready to handle fetches!');
  });

  self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  // serve the cat SVG from the cache if the request is
  // same-origin and the path is '/dog.svg'
  if (url.origin == location.origin && url.pathname == '/dog.svg') {
  event.respondWith(caches.match('/cat.svg'));
  }
  });

  ```

- clients.claim

  service worker 激活后，您可以在 service worker 中调用 `client.claim()` 来控制不受控制的客户端。

  这是上述 demo 的[变体](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/df4cae41fa658c4ec1fa7b0d2de05f8ba6d43c94/)，是在其 activate 事件中调用 client.claim（）的示例。您应该第一次见到猫。我说“应该”，因为这是对时间敏感的。 如果 service worker 激活并且 client.claim（）在图像尝试加载之前生效，您只会看到一只猫。

  ```code
  self.addEventListener('activate', event => {
    clients.claim();
    console.log('Now ready to handle fetches!');
  });
  ```

  如果您使用 service worker 加载的页面与通过网络加载的页面不同，则 client.claim（）可能会很麻烦，因为 service worker 最终将控制一些未使用它加载的客户端([原文](https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle#clientsclaim))。

- 缓存并返回请求

  在安装 service worker 并用户导航到其他页面或刷新后，service worker 将开始接收 fetch 事件，下面是一个示例。

  ```code
  self.addEventListener('fetch', function(event) {
    event.respondWith(
      caches.match(event.request)
        .then(function(response) {
          // Cache hit - return response
          if (response) {
            return response;
          }
          return fetch(event.request);
        }
      )
    );
  });
  ```

  在这里，我们定义了 `fetch` 事件，在 `event.respondWith()` 中，我们从 `caches.match()` 中传递了一个 promise。此方法查看 request，并从您的 service worker 创建的任何缓存中查找所有缓存的结果。

  如果匹配响应(response)，则返回缓存的值，否则返回对 `fetch` 的调用的结果，这将发出网络请求，并返回数据。这是一个简单的示例，它使用了我们在安装步骤中缓存的所有缓存资源。

  如果我们要累积缓存新请求，可以通过处理获取请求的响应，然后将其添加到缓存中来实现，如下所示。

  ```code
  self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        // Cache hit - return response
        if (response) {
          return response;
        }

        return fetch(event.request).then(
          function(response) {
            // Check if we received a valid response
            if(!response || response.status !== 200 || response.type !== 'basic') {
              return response;
            }

            // IMPORTANT: Clone the response. A response is a stream
            // and because we want the browser to consume the response
            // as well as the cache consuming the response, we need
            // to clone it so we have two streams.
            var responseToCache = response.clone();

            caches.open(CACHE_NAME)
              .then(function(cache) {
                cache.put(event.request, responseToCache);
              });

            return response;
          }
        );
        })
      );
  });
  ```

  我们正在做的是这样的：

  - 在 fetch 请求上给 `.then()` 添加一个回调。
  - 得到响应后，我们将执行以下检查：
    - 确保响应有效。
    - 检查响应的状态为 200。
    - 确保响应类型是 basic ，它表明该请求来自我们的源(origin)。这意味着对第三方资源的请求也不会被缓存。
  - 如果我们通过检查，我们将`克隆响应`。这样做的原因是因为响应是流，所以 body 只能被使用(consumed)一次。由于我们要返回响应以供浏览器使用，并将其传递给缓存以使用，因此我们需要对其进行克隆，以便将其发送到浏览器，并将其发送到缓存。

### fetch 的默认行为

- No credentials

  当您使用 fetch 时，默认情况下，请求不会包含 `cookie` 之类的凭据。如果你想要凭据，那就调用:

  ```code
  fetch(url, {
    credentials: 'include'
  })
  ```

  这种行为是有意为之，而且可以说比 XHR 的更复杂的默认发送凭证(如果 URL 是同源则发送，但是在其他情况下则会忽略)要好。fetch 的行为更像其他 CORS 请求，如 `<img crossorigin>`，它永远不会发送 cookie，除非你选择与`<img crossorigin="use-credentials">`。

- Non-CORS fail

  默认情况下，从不支持 CORS 的第三方 URL fetch 资源将会失败。您可以在请求中添加 `no-CORS` 选项以解决此问题，尽管这将导致“不透明”响应，这意味着您将无法确定响应是否成功。

  ```code
  cache.addAll(urlsToPrefetch.map(function(urlToPrefetch) {
    return new Request(urlToPrefetch, { mode: 'no-cors' });
    })).then(function() {
    console.log('All resources have been fetched and cached.');
  });
  ```

### 处理响应图像

`srcset` 属性或 `<picture>` 元素将在运行时选择最合适的图片资源并发出网络请求。

对于 service worker，如果您想在安装步骤中缓存图片，则有以下几种选择：

1. 安装 `<picture>` 元素和 `srcset` 属性将请求的所有图片。
2. 安装单个低分辨率版本的图片。
3. 安装单个高分辨率版本的图片。

实际上，您应该选择选项 2 或 3，因为下载所有图片会浪费存储空间。

假设您在安装时选择了低分辨率版本，并且想要在加载页面时尝试从网络中检索较高分辨率的图片，但是如果高分辨率图片失败，则回退到低分辨率版本。 这样做很好，但是有一个问题。如果我们有以下两个图片：

| Screen Density | Width | Height |
| -------------- | ----- | ------ |
| 1x             | 400   | 400    |
| 2x             | 800   | 800    |

在 `srcset` 图像中，我们有如下标记：

```html
<img src="image-src.png" srcset="image-src.png 1x, image-2x.png 2x" />
```

如果我们使用的是 2x 显示屏，那么浏览器将选择下载 image-2x.png，如果我们处于离线状态，则可以 .catch（）此请求并返回 image-src.png（如果已缓存），但是浏览器会期望图片考虑到 2x 屏幕上多余像素，因此该图像将显示为 200x200 CSS 像素，而不是 400x400 CSS 像素。 解决此问题的唯一方法是在图像上设置固定的高度和宽度。

```html
<img src="image-src.png" srcset="image-src.png 1x, image-2x.png 2x" style="width:400px; height: 400px;" />
```

对于 `<picture>` 元素，这将变得更加困难，并且在很大程度上取决于图像的创建和使用方式，但是您可以使用类似的方法来设置 srcset。

## 更新 Service Worker

简而言之:

- 如果发生以下任何情况，将触发更新：

  - 导航到作用域内(in-scope)页面
  - push 和 sync 之类的功能性(functional)事件, 除非在过去 24 小时内进行过更新检查
  - 调用 `.register()` (仅当 service worker URL 更改时) 。然而，您应该避免更改 service worker URL。

- 大多数浏览器（包括 Chrome 68 和更高版本）在检查注册的 Service Worker 脚本的更新时会`默认忽略缓存头部`。当通过 `importScripts()` 获取加载在 service worker 内部的资源时，它们仍然尊重缓存头部。您可以通过在注册 service worker 时设置 `updateViaCache` 选项来覆盖此默认行为。
- 如果您的 service worker 与浏览器已有的`字节不同(byte-different)`，则认为它已更新。 （我们正在将其扩展为也包括导入的脚本/模块。）
- 更新后的 service worker 与现有的 service worker 一起启动，并获得自己的 install 事件。
- 如果新的 service worker 的状态码不正确（例如 404），无法解析，在执行过程中抛出异常或在安装过程中 reject，则新的 service worker 将被丢弃，但当前 service worker 仍处于活动状态。
- 成功安装后，更新后的 service worker 将等待，直到现有 service worker 控制零个客户端为止。 （请注意，客户端在刷新期间会重叠。）
- `self.skipWaiting()` 避免了等待，这意味着 service worker 在完成安装后立即激活。

### 更新过程

- 安装

  请注意，我已将缓存名称从 static-v1 更改为 static-v2。 这意味着我可以设置新的缓存，而不会覆盖当前缓存(旧 service worker 仍在使用该缓存)。

  这种模式创建特定版本的缓存，类似于本机应用程序与其可执行文件捆绑在一起的资产。您可能还具有非特定版本的缓存，例如头像。

- 等待(Waiting)

  成功安装后，更新的 service worker 会`延迟激活`，直到现有的 service worker 不再控制客户端为止。此状态称为“等待”，这是浏览器确保每次仅运行一个版本的 Service Worker 的方式。

  如果您运行更新的 [demo](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/index-v2.html)，则仍应看到猫的照片，因为 V2 service worker 尚未激活。 您可以在 DevTools 的“应用程序”选项卡中看到新的 service worker 正在等待：

  ![waiting](https://developers.google.com/web/fundamentals/primers/service-workers/images/waiting.png)

  即使您只用一个选项卡打开 demo，刷新页面也不足以让新版本接管。这是由于浏览器导航的工作原理导致的。导航时，当前页面不会消失，直到收到响应头(response headers)，并且如果收到的响应包含 Content-Disposition 头部, 当前页面也可能会停留。由于存在这种`重叠`，当前的 service worker 始终在刷新期间控制客户端。

  要获取更新，请使用当前 service worker 的`所有 tab 关闭`或`导航至别处`。然后，当您再次导航到 demo 时，您应该会看到这匹马。

  此模式类似于 Chrome 更新的方式。Chrome 的更新会在后台下载，但要等到 Chrome 重新启动后才能应用。同时，您可以继续使用当前版本而不会中断。但是，这在开发过程中很痛苦，但是 DevTools 可以使它变得更容易，我将在本文后面介绍。

- 激活(Activate)

  一旦旧的 service worker 消失，并且新的 service worker 能够控制客户端，它就会触发。这是完成旧 worker 仍在使用时无法完成的任务的理想时机，比如迁移数据库和清除缓存。在上面的演示中，我维护了一个我希望存在的缓存列表，并且在 activate 事件中，我删除了任何其他缓存(这将删除旧的 static-v1 缓存)。

  注意: 您可能不是从之前的版本更新的。因为一个 service worker 可能存在很多个旧版本。

  如果你传递一个 promise 给 event.waitUntil()，它会`缓冲(buffer)`功能性事件(fetch, push, sync 等)，直到 promise 被 resolve。因此，当 fetch 事件触发时，激活就完全完成了。

  警告：缓存存储 API 是“`源存储(origin storage)`”（例如 localStorage 和 IndexedDB）。如果您在相同的来源(origin)上运行许多站点（例如，`youname.github.io/myapp`），请注意不要删除其他站点的缓存。为避免这种情况，请为您的缓存名称提供当前站点唯一的前缀，例如 myapp-static-v1，并且不要接触缓存，除非它们以 myapp-开头。

- 跳过等待阶段

  等待阶段意味着您一次只能运行站点的一个版本，但是如果您不需要该功能，则可以通过调用 `self.skipWaiting()` 使新的 service worker 更快地激活。

  这会使您的 service worker 踢出(kick out)当前活动的 worker，并在进入等待阶段时立即激活自己（如果已经进入等待阶段，则立即激活）。它不会导致您的 service worker 跳过安装，而只是跳过等待。

  何时调用 skipWaiting（）并不重要，只要它在等待期间或等待之前即可。在 install 事件中调用它是很常见的：

  ```code
  self.addEventListener('install', event => {
    self.skipWaiting();

    event.waitUntil(
      // caching etc
    );
  });
  ```

  但您可能希望将它作为一个 postMessage() 到 service worker 的结果调用。例如，您希望在用户交互之后 skipWaiting（）。

  这是一个使用 skipWaiting（）的 [demo](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/index-v3.html)。您应该不必走开(navigate away)就能看到一头母牛的照片。 像 clients.claim（）一样，这是一场竞赛(race)，因此只有在新的 service worker 在页面尝试加载图像之前获取，安装并激活后，您才能看到这头牛。

  警告：skipWaiting（）意味着您的新 service worker 可能会控制由旧版本加载的页面。这意味着您的某些页面的 fetches 已经由您的旧 service worker 处理过了，但是您的新 service worker 将处理后续的 fetches。如果这可能会破坏事情，请不要使用 skipWaiting（）。

### 手动更新(Manual updates)

如前所述，浏览器会在导航和功能性事件后自动检查更新，但您也可以手动触发它们：

```code
navigator.serviceWorker.register('/sw.js').then(reg => {
  // sometime later…
  reg.update();
});
```

如果您希望用户长时间使用您的站点而不需要重新加载，那么您可能希望每隔一段时间(比如每小时)调用 update()。

### 避免改变 service worker 脚本 URL

如果您已经阅读了我关于缓存最佳实践的[文章](https://jakearchibald.com/2016/caching-best-practices/)，您可能会考虑为 service worker 的每个版本提供唯一的 URL。 不要这样！对于 service worker 来说，这通常是不好的做法，只需在其当前位置更新脚本即可。

它可能使您遇到这样的问题：

1. index.html 将 sw-v1.js 注册为 service worker。

2. sw-v1.js 缓存并提供 index.html，因此它可以离线优先运行。

3. 您更新 index.html，以便它注册新的 sw-v2.js。

如果执行上述操作，则用户永远不会获得 sw-v2.js，因为 sw-v1.js 正在从其缓存中提供旧版本的 index.html。 您已将自己置于需要更新 service worker 以更新 service worker 的位置。

但是，对于上面的 demo，我更改了 service worker 的 URL。 这样，就 demo 而言，您可以在版本之间进行切换。这不是我在生产中要做的。

```code
// 更新 service worker 的代码
const expectedCaches = ['static-v2'];

self.addEventListener('install', event => {
  console.log('V2 installing…');

  // cache a horse SVG into a new cache, static-v2
  event.waitUntil(
    caches.open('static-v2').then(cache => cache.add('/horse.svg'))
  );
});

self.addEventListener('activate', event => {
  // delete any caches that aren't in expectedCaches
  // which will get rid of static-v1
  event.waitUntil(
    caches.keys().then(keys => Promise.all(
      keys.map(key => {
        if (!expectedCaches.includes(key)) {
          return caches.delete(key);
        }
      })
    )).then(() => {
      console.log('V2 now ready to handle fetches!');
    })
  );
});

self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  // serve the horse SVG from the cache if the request is
  // same-origin and the path is '/dog.svg'
  if (url.origin == location.origin && url.pathname == '/dog.svg') {
    event.respondWith(caches.match('/horse.svg'));
  }
});
```

### 开发工具的使用

参见 [Making development easy](https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle#devtools)

参考资料

[Service Workers: an Introduction](https://developers.google.com/web/fundamentals/primers/service-workers)

[The Service Worker Lifecycle](https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle)

[Service Worker Registration](https://developers.google.com/web/fundamentals/primers/service-workers/registration)

[High-performance service worker loading](https://developers.google.com/web/fundamentals/primers/service-workers/high-performance-loading)
