# 《初识Service Worker及简单应用》

## 前言
在移动当道的今天，人们对端的体验越来越高。web虽然拥有快速发布、动态部署等特性，但相对于native来说，体验还是差了一截。开发者们在webapp的开发中做了大量的优化尝试，其中，Progress Web App (渐进式Web应用)被广泛提及并逐渐被大家认可。今天我们就对PWA中核心的一个技术点 `service worker` 来和大家做探讨，希望能对它有个更深入的了解。

## 什么是 Service Worker？
service worker 也称服务工作线程，是浏览器在后台独立网页运行的脚本，也算作是 Javascript 工作线程。它无法直接访问 DOM，因此，如果你需要操作页面的 DOM 节点的话，可以通过 [postMessage](https://html.spec.whatwg.org/multipage/workers.html#dom-worker-postmessage) 来跟想控制的页面进行通信。 service work 中的 API 大量采用 Promise 方式设计，因此代码比较友好。

在兼容性方面， Chrome Firefox Opera 都已经支持， Microsoft Edge 现在也表示公开支持。而之前 Safari 因为不计划支持被很多开发者吐槽，认为它将会是下一代 IE 。迫于压力下，现 Safari 也[暗示未来会进行开发](https://trac.webkit.org/wiki/FiveYearPlanFall2015)。

### 使用前提
如果网站要使用 service worker ，传输协议必须为 HTTPS 。因为 service worker 中会涉及到请求拦截，所以必须使用 HTTPS 协议来保证安全。 另外，如果需要本地调试 service worker 的话， `localhost` 是被支持的。

## Service Worker 能做什么？
以下罗列了几点当前以及将来 Service Worker 能做的事情：

- 拦截网络
- 离线缓存
- 消息推送
- 后台同步
- 定期同步（future support）
- 地理围栏（future support）

> 地理围栏（Geo-fencing）是 LBS 的一种新应用，就是用一个虚拟的栅栏围出一个虚拟地理边界。当手机进入、离开某个特定地理区域，或在该区域内活动时，手机可以接收自动通知和警告。

## 利用 Service Worker 构建离线应用

### Service Worker 的生命周期
<img src="https://img.alicdn.com/tfs/TB10yUxSFXXXXXkXXXXXXXXXXXX-702-685.png" width="600" />

#### 注册 `register`
当你的应用之前未注册过 service worker 的话，那么第一步将会是注册环节：

```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js').then((registration) => {
      // succses
    }).catch((err) => {
      // error   
    })
  })
}
```
以上代码有几个点需要注意：

- 需要进行特性检测，判断浏览器是否支持。
- 最好在页面所有资源都已经加载完毕后，这个时候去加载 service worker 是一个非常好的时间点。因为在移动端，页面打开首屏时间非常关键。

**请注意，以上 '/sw.js' 代表 service worker 作用域的范围为根域，也就是说，即使你的页面在 '/example/a.html' 中也属于该 service worker 的控制范围**。

Chrome 浏览器下，可以在 `chrome://inspect /#service-workers` 中，查看服务工作线程是否已经注册。如果调试的话，用隐身模式打开窗口会非常方便，因为从隐身窗口创建的任何注册和缓存在该窗口关闭后均将被清除

#### 安装 `install`
在受控页面注册完成后，我们就需要处理 service worker 的 `install` 事件。  
通常在这个回合里，我们会缓存文件。

```js
self.addEventListener('install', function(event) {
  // 进行安装回调
});
```

接下来在回调里缓存文件：
1.打开缓存空间。
2.缓存文件。

```js
var CACHE_NAME = 'my-cache-v1';
var urlsToCache = [
  '/',
  '/styles/base.css',
  '/dist/bundle.js'
];

self.addEventListener('install', (event) => {
  // 开始缓存文件
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(function(cache) {
        console.log('成功打开缓存空间');
        return cache.addAll(urlsToCache);
      })
  );
});
```

请注意，这里面有个缓存空间的概念，也即 `caches.open` 里的参数。因为每个页面对应的缓存空间可能不相同，有了缓存空间后也能更好的对缓存进行控制。打开缓存空间后，之后再调用 `cache.addAll()` 并传入文件数组进行缓存。

上述中，所有的文件都成功缓存才算成功，如果有任何文件下载失败，那么安装步骤就会失败。所以如果缓存列表过长的话，将会增加缓存失败的几率。

**如果sw.js一直都不变化的话，那么 install 事件只有在首次安装的时候才会调用。**

#### 响应请求 `fetch`
假如我们成功安装了 service worker ，那么在用户下次再次进入页面的时候，我们就要把已缓存的文件返回。
当用户刷新页面或者跳至子域页面的时候，service worker 会触发 `fetch` 事件。比如：

```js
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((response) => {
        // 如何有缓存的话，那么就直接返回缓存，否则直接获取源文件
        if (response) {
          return response;
        }
        return fetch(event.request);
      }
    )
  );
});
```

上述是一个简单的例子：有缓存的话直接返回缓存文件，没缓存的话获取源文件并返回。  
如果做的好的话，可以在发现没缓存的时候，把它先缓存下来，再进行返回。

#### 更新
如何更新 `sw.js` 及 如何更新缓存呢？
浏览器一旦判断到新的服务工作线程文件与其当前所用文件存在字节差异，则将其视为“新服务工作线程”。接下来就会做以下事情：
1.新的服务工作线程启动，并触发 `install` 事件。
2.此时，旧服务工作线程仍控制着当前页面，因此新服务工作线程将进入 `waiting` 状态。
3.当网站上当前打开的页面关闭时，旧服务工作线程将会被终止，新服务工作线程将会取得控制权。
4.新服务工作线程取得控制权后，将会触发其 `activate` 事件。

通常在 `activate` 事件中，我们会清除旧版本的缓存。

```js
self.addEventListener('activate', (event) => {
  const cacheWhitelist = ['my-cache-v2'];

  event.waitUntil(
   // 找到所有老的缓存空间
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames.map((cacheName) => {
          // 如果不在当前缓存空间白名单列表中就删除。
          if (cacheWhitelist.indexOf(cacheName) === -1) {
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
});
```

## 注意事项
在 service worker 中使用 `fetch` 的时候，请求头中默认不包含 `Cookie`，如果要添加 `Cookie` 的话，需要增加 credentials 参数。

```js
fetch(url, {
  credentials: 'include'
})
```

## 总结
除了离线缓存外，利用 service worker 还可以做很多令人兴奋的功能。比如：

- 客户端编译（把原来本地/服务端webpack打包编译的事情放在client中）
- 预请求Prefetch（ssr框架 next.js 提供了Link组件来实现预请求渲染）

利用 service worker 能大大提升 Progress Web App 的体验，现在各大厂商都表示支持，也代表了未来的趋势。另外， Google 提供了一套 Service Worker 库，可以消除服务工作线程样板文件代码，从而简化开发工作。

- [sw-precache](https://github.com/GoogleChrome/sw-precache/)—与构建流程集成，以生成服务工作线程，用于预缓存静态资产，例如 Application Shell。
- [sw-toolbox](https://github.com/GoogleChrome/sw-toolbox/)—实现常见运行时缓存模式，例如动态内容、API 调用以及第三方资源，实现方法就像编写 README 一样简单。
- sw-offline-google-analytics—临时保留并重试 Analytics 请求，以避免请求因网络断开连接而丢失。

<img src="https://img.alicdn.com/tfs/TB1BwvPSFXXXXX9aXXXXXXXXXXX-690-720.gif" width="400" />

## 参考文档

- [Service Workers: an Introduction](https://developers.google.com/web/fundamentals/getting-started/primers/service-workers)
- [service worker libraries](https://developers.google.com/web/tools/service-worker-libraries/)
- [Using Service Workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers)
