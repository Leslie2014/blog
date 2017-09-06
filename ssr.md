# 关于Node服务端渲染
![](https://img.alicdn.com/tfs/TB1RckHb5ERMeJjSspjXXcpOXXa-700-263.png)
> 图片源自[The Benefits of Server Side Rendering Over Client Side Rendering](https://medium.com/walmartlabs/the-benefits-of-server-side-rendering-over-client-side-rendering-5d07ff2cefe8)文章

## 什么是服务端渲染
服务端渲染也称作 `SSR(Server Side Render)` 。不同于客户端渲染，服务端渲染会在后端把页面DOM的结构树转成 String 吐出来，然后到前端（如浏览器）解析渲染。

## 优势

### SEO
现在单页面应用由于体验好，广泛流行。但单页应用的做法往往是后端只吐出一个页面的框架，里面没有具体内容，然后前端通过 Ajax 动态拉取内容。这就导致爬虫去访问你的站点时，服务端返回给爬虫的只有一个架子，爬虫无法抓取页面关键词之类等信息。

### 首屏直出
意思很好理解，就是在用户首次访问的时候不用再看到菊花在那里转呀转(Loading...)，首屏就可以看到页面所有内容。另外可以在服务端通过 HTTP 接口合并请求等方式，让页面打开的首屏时间缩短。

## Node 服务端渲染有什么特别？
同构（isomorphic）！我想这个应该是用Node做服务端渲染最大的优势。那么什么是同构呢？

其实同构大多是由 isomorphic 单词翻译来的，这个单词比较难理解含义，现在也有很多叫做 universal app。意思差不多，就是说能够实现一套代码在服务端跟客户端同时运行。

假如我们客户端的页面是 React 写的，那么这套代码也能在服务端运行，并进行渲染，这就是同构的概念。同一份代码，运行构建于两端。因为都是 javascript 语法，所以用 Node 做服务器渲染在这方面有天生的优势。

## 用 Node 做同构有什么难点？

### 运行环境支持
现在的前端开发，大多数用的是ES6/7的语法，然后用 Babel 编译成 ES5/3后让浏览器运行。Node对 ES6/7 的支持并不是十分友好，就算是最新版本的 node，也不还是不支持ES Module（就是我们常看到的 `import` 语法引入模块）。所以要达到同一份代码两端运行的目的。就必须磨平运行环境的差异。

那么该如何做呢？答案就是 [babel-register](https://babeljs.io/docs/usage/babel-register/)。

babel-register 模块会改写 require 命令，为它加上一个钩子。此后，每当使用require加载 .js 、 .jsx 、 .es 和 .es6 后缀名的文件，就会先用 Babel 进行转码。当然，这就要求你必须在你服务端入口文件的顶部率先加载这个模块。

### 资源加载

如果说磨平环境差异还不算困难，那让Node支持多种资源类型加载估计是要让你头皮发麻了。
比如说我们现在用 react 开发app，app中必然涉及到 css/scss 、 png/jpg 、 font 等文件的加载吧？我们一般是通过webpack的loader来处理的。那这段代码运行在服务端会怎么样？必然是血崩。。。

node `require` 默认就只支持加载 .json .js 等几种文件，所以如何保证客户端渲染出来的代码跟服务端渲染出来的代码一致呢（在 react 应用中，react 会检查客户端渲染出来的结构是否跟客户端渲染出来的一致，如果不一致的话，会在客户端重渲染）？这里提供两套思路：

1. 客户端跟服务端用同一套 webpack 打包后的资源。[webpack-isomorphic-tools](https://www.npmjs.com/package/webpack-isomorphic-tools) 可以很好的解决这个问题，或者最新的 webpack 版本 `target: node` 也能实现。

2. png/jpg/font 等文件直接忽略（在 babel-register 里可以设置），scss/css的话，用 `css in js` 的方式写。

## 总结
Node服务端渲染好处多多，但除了上述技术性的问题需要解决外，仍然有些线上问题需要注意。

首当其冲的就是服务器 cpu 过高问题，因为现在页面结构是在服务端以 `renderToString` 的方式输出，所以页面请求路由会涉及到大量的计算。这就会导致如果页面并发高一点的话，会出现 cpu 过高的问题。

另外在服务端可没有什么 window 、 document 对象，这些东西也需要去 hook 掉；在 React 应用中，componentDidMount 等生命周期函数也不会在服务端触发；定时器记得及时释放，否则可能会导致内存泄露的风险！

如果你确定要用 node 做服务端渲染的话，建议你应该用一些开源成熟的框架。比如在 react 体系下比较有代表性的 [next.js](https://github.com/zeit/next.js/)， vue 体系下的 [Nuxt.js](https://nuxtjs.org/)。


