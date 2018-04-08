# 基于 React 的单页应用优化技巧

本文内容是基于项目前端框架优化而提炼出的一些比较实用的单页优化小技巧，在此做一下沉淀，希望能帮到在这方面摸索的同学。

本人前端框架是在 React 16+、react-router v4、webpack v3、Next1.x 之上构建。 Next 是阿里集团电商通用 UI 组件库（与 Antd 类似）。下面开始介绍一些常用的单页应用优化技巧。

## 路由设计

单页应用，顾名思义，就是只有一个 HTML 文档保存在服务器，服务器吐出该文档后，之后的一切操作都在前端进行（当然，数据还是通过 Ajax 异步调用接口拿取的）。那么问题来了，如何保证切换页面时，页面 path 改变且不刷新页面呢？

我们传统的做法是把页面路由设计放在后端，这样请求一个页面路径，服务器返回文档，浏览器刷新。这种玩法对普通的应用没什么问题，但对单页应用显然不合适，因为如果页面的框架固定的话（如Header、侧边栏），每次为了改变其中的内容而刷新页面，会导致不好的用户体验。因此，路由设计这块得放前端了。

前端动态的切换子页面，有一种方式是通过更换锚点。如 `www.taobao.com/#a` 和 `www.taobao/com/#b` 前端框架通过识别锚点来切换预先设置好的页面。这从功能层面上来说好像没什么问题，不过，如果你有点追求的话，你应该不会喜欢这种路由方式，因为这不够语义化，不够直观。比如我要加一个“新增类目”的页面，那么你应该想到的是`/category/add`，而不是 `#addCategory`。

那么如何实现修改页面 path 而不刷新页面（请求服务器）呢？HTML5 的 `History` API 可以去了解一下。简单来说，可以通过这个 API 的 `pushState` 和 `popState` 来人为的操控浏览器历史记录。

好了，这个时候你可能沉浸在页面无缝切换带来的快感。然后一不小心点了下刷新，GG。为什么就挂了呢？那是因为你之前对浏览器前进后退并没有走服务器请求，但是刷新页面还是会走服务器请求的，这个时候后端又没有设计对应路径的路由，当然就挂了。所以，为了修复这个问题，服务器应该做一点逻辑处理，那就是对未匹配到的路由一律返回那唯一的 HTML 文档。你可以会问那 404 咋办？这让前端处理下就好了。或者更方便的处理方式是直接在 nginx 层做以下配置：

```
location / {
  try_files $uri /index.html;
}
```

## Code Split & Loading过渡

路由结构搭好了后，然后开始疯狂的写 js (bug)。在你以为可以下班回家后，最后进行前端资源打包，GG。你默默的告诉自己，一个 JS 3M，Gzip 压一下估计也就 900K 了，现在网速这么快，问题不大。在你准备起身的时候，你想起了做前端的初心。倒了杯热水后，继续加班。

900K 的 JS 在弱网环境下，加载还是要个 2-3 秒的，此时用户的页面将会是空白。于是，我们要对这个 JS 做拆分，能不能先加载页面框架所需的那部分通用的 JS 代码，然后再加载具体页面的 JS 呢？Code Split 就是干这事情的。传统的 Code Split 是拆分出多个独立页面中通用的 JS，然后每个页面里都加载通用+独立的 JS。这种方式在单页应用里走不通。

单页应用要做 Code Split 的话，就需要运用动态加载方案（dynamic import）。即页面一开始只加载初始化的 JS（内容包含JS框架+通用的组件），然后根据具体的页面加载具体的 JS 代码。webpack2.0 以下版本可以使用 `require.ensure` 可以实现，webpack2.4 以上版本可以使用所谓的 `magic comment` 实现：`import(/* webpackChunkName: "my-chunk-name" */ 'module');`。

为了体验友好，在页面 js 未完全加载完之前，应该有个 Loading 动画过渡。为了快速实现以上两点，可以使用现有完备的解决方案 [react-loadable](https://github.com/jamiebuilds/react-loadable)。

以上两点都实现后，在本地环境测试也没问题，然后发布到了线上，GG，心态大崩。你发现拆分出的 JS bundle 加载的路径是错误的：页面的域名是`www.taobao.com`，主 bundle 的地址是 `g.alicdn.com/a/build123`，而加载的拆分子 bundle 的地址确实 `www.taobao.com/build456`。在擦干眼角的泪水后，你打开了著名的堆栈溢出网 `stackoverflow`。在参考有关资料后，你发现资源的加载路径跟 webpack 中的 `publichPath` 路径有关，于是你修改了针对线上的 webpack 配置，重新发布上线。在看到 JS 正常加载后，流下了激动了泪水。

## Reducer 动态注入

由于项目是基于 Redux 来做应用状态管控的。Redux 的特点是比较啰嗦，但是对团队协作，规范约定还是比较不错的（redux saga 什么之类的也考虑用，不过项目时间紧，让外包同学去临时学习时间成本高）。因为啰嗦，所以意味着代码量也上去了，如果页面主 bundle 中一开始就把所有页面的 action reducer 都加载进来的话，那主 bundle 又变大了。且为了优化性能，并不想项目初始化的 root state 就包含了所有页面的初始状态，想要的效果是某个子页面加载完毕后，再向根 state 注入。于是有了下面这段 reducer 注册代码，主要用了发布订阅模式：

```javascript
export class ReducerRegistry {
  constructor() {
    this._emitChange = null;
    this._reducers = {};
  }
  // 返回当前应用的 reducer
  getReducers() {
    return { ...this._reducers };
  }
  /**
   * 注册 reducer
   * @param {String} name reducer的名称
   * @param {Function} reducer reducer 对象
   */
  register(name, reducer) {
    this._reducers = { ...this._reducers, [name]: reducer };

    if (this._emitChange) {
      this._emitChange(this.getReducers());
    }
  }

  setChangeListener(listener) {
    this._emitChange = listener;
  }
}

const reducerRegistry = new ReducerRegistry();
export default reducerRegistry;
```

然后在主 bundle 中监听注入事件，并替换 store reducer：

```javascript
reducerRegistry.setChangeListener(reducers => {
    store.replaceReducer(combineReducers(reducers));
});
```

在子页面中注入 reducer：

```javascript
reducerRegitry.register("pageName", reducer);
```

以上，就完成了 reducer 动态注入功能。

更多的技巧，欢迎留言讨论。


