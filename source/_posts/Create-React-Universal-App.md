title: 前后端同构之路
date: 2016-10-24
categories: 架构分析
tags: [react, Javascript, universal]
toc: true

---

公司内部的公用图标字体平台 iconfont 上线一年以来，运行良好，在内部承担起了客户端减容、提高前端开发效率的任务。但由于其臃肿的架构导致的运行效率底下和页面卡顿等问题也一直给使用者造成诸多不便。我们对项目老架构导致的许多问题进行了探索总结，并针对问题提出了 react 同构的解决方案。

<!--more-->

## 旧版架构问题探索

在进行旧版平台开发的时候，我们采用了当时淘宝推出的先进思想：[中途岛 midway 前后端分离方式](https://www.zhihu.com/question/23512853)，搭建了一个 node 中间层进行页面的渲染，以求提升页面的渲染速度。我们旧版平台的结构如下：

![旧版平台的三层结构](http://s.qunarzz.com/iconfont/blog/universal-app/old-architecture.png)

从图中我们可以看到，尽管我们前端掌握了 server，可以进行页面渲染的控制，但是服务端的渲染和前端的渲染和路由依然是割裂的，之间有很多冗余的内容。导致这些冗余的主要原因，其实还是前后端渲染方式不一致以及前后端代码的分离。

### 是否需要前后端分离

我们知道，在传统的 MVC 架构的项目之中，js 代码只占 View 层的很小的一部分。随着项目的渐进发展，前端功能的复杂度日益增高，导致项目难以维护；同时前后端语言并不一致（我们都知道 Java 跟 JavaScript 基本上是雷锋和雷峰塔的关系），不同的开发在一个项目里操作极为不便，因此才产生了前后端分离。

但是随着 js 向服务端的进发，我们的中间层 server 也采用支持 js 的 node 来进行架构，所以前后端语言不一致的问题基本上抹平了；而前端功能复杂这一点，从刚才的分析我们也可以看到，其实前端和后端在路由、渲染这些功能上是有很大的重合，因此前端的 server 和前端逻辑项目没有必要进行分离。

实际上，这里我们的前后端分离，已经有传统意义上前端和后端代码的分离、服务端和浏览器客户端的分离，演变为后端数据提供和前端提供渲染的分离。

### 前后端混合渲染的问题

如果将前后端代码糅合在一起，那么渲染这里将会是服务端逻辑和客户端逻辑的一个结合点，它们的模板、渲染方式都一定要一致，才能减少开发的工作量。

对于我们旧版项目来说，服务端采用 handlebars 作为模板，而前端采用 MVVM 模式的 avalon 的模板，两者在用法和理念上都是有一定冲突的。其中 MVVM 模式在服务端渲染中最棘手的问题就是：**要实现双向数据绑定，必须要经历一次 DOM 渲染**。这样就导致后端只能渲染一个中间状态的模板，然后还需要前端在更改一次 DOM，无法达到『直出』的效果。

![前后端渲染模板差异](http://s.qunarzz.com/iconfont/blog/universal-app/back-to-front.png)

这个问题看似困难，但在 react 出现之后，却得到了完美的解决：react 基于 virtual DOM，不需要扫描 DOM 来建立双向绑定关系，只需要在每次状态变动时进行 diff，有变化才会进行更新。因此，我们可以在服务端直接渲染出 DOM 结构，如果前端最终生成的虚拟 DOM 跟后端直出的 DOM 保持一致，那么就不需要更改 DOM 结构，大幅度提升渲染速度。

## 同构应用的构建

如果要实现前后端代码同构，其实只要保证两个一致即可：**包管理工具**和**模块依赖方式**的一致。这里我们可以看到，这二者的一致性都能得以保证：

1. 包管理工具：前端和 node 目前都采用 npm 来进行依赖管理，这就保证客户端和服务端都可以使用同一个兼容包；
2. 模块依赖方式：通过 webpack 这样的打包工具，可以保证前端和均采用 CommonJS 的依赖方式，确保代码可以互相依赖。

有了这二者的保证，我们就可以完美的解决同构的问题，剩下需要考虑的就是如何处理服务端渲染了。

## react 全家桶对服务端渲染的支持

react 自诞生之初就对服务端渲染非常重视，它的『全家桶』都对服务端渲染进行了良好的支持。所谓的『全家桶』指的就是大家耳熟能详的 react 御三家：[react](https://facebook.github.io/react/)、[react-router](https://github.com/ReactTraining/react-router) 和 [redux](http://redux.js.org/)。

![react 全家桶](http://s.qunarzz.com/iconfont/blog/universal-app/react-everything.png)

### React 和 ReactDOM

ReactDOM 在这里提供的支持就是 `ReactDOM.render` 和 `ReactDOM.renderToString` 函数，其中前者会在浏览器端生成 DOM 结构，后者会在服务端生成对应的 HTML 字符串模板。React 会在生成的 DOM 结构上添加一个 `data-react-checksum` 的属性，这是一个 adler32 算法的校验和，用以确保两份模板的一致性。

![checksum 判断](http://s.qunarzz.com/iconfont/blog/universal-app/checksum.png)

同时 react 的生命周期在前后端渲染过程中也有所不同。前端渲染的组件拥有完整的生命周期，而后端渲染仅有 `componentWillMount` 的生命周期。这就意味着，如果我们想进行前后端共同操作的逻辑，如发送数据请求等，可以放在 `componentWillMount` 的生命周期中；如果想单独处理客户端的逻辑，可以放在其他生命周期，如 `componentDidMount` 中。

### react-router

react-router 是 react 的路由 - 视图控制库，可以书写便捷的声明式路由以控制不同页面的渲染。react-router 本身是一个状态机，根据配置好的路由规则，和输入的 url 路径，通过 `match` 方法找到对应的组件并进行渲染。

![react-router 原理](http://s.qunarzz.com/iconfont/blog/universal-app/react-router.png)

这套机制在前端和后端都是相通的，例如在后端，就是下面这样一种实现形式来进行渲染：

```js
app.use(async (ctx, next) => {
  match({
    location: ctx.originalUrl,
    routes
  }, callback)
  // 渲染完成之后，调用 callback 回调
  // 将 <RouterContext> 组件 renderToString 返回前端即可
})
```

对于前端来说，其实也是处理的上面这些逻辑，不过它被很好的封装在 `<Router>` 组件中，我们只需要写好声明式的路由，这一切就可以随着 url 的变化自动发生。

### redux

redux 是 react 的数据流管理库，它对服务端渲染的支持很简单，就是**单一 store**和**状态可初始化**。后端在进行渲染的时候会构建好单一的 store，并将构建好的初始状态通过以 JSON 格式，通过全局变量写到生成好的 HTML 字符串模板上。前端通过获取初始状态，生成跟后端渲染完成后一模一样的 store，就可以保证前后端渲染数据的一致，以确保前后端生成 DOM 结构的一致。

## 项目优化成果总结

项目在使用了 react 进行同构构建之后，首屏渲染性能得到了明显的提升，之前页面大约 1500ms 才能展示关键数据，而之后的页面只需要 230ms 就可以进行数据展示了。

![性能对比](http://s.qunarzz.com/iconfont/blog/universal-app/render.png) 

同时项目得到了精简，由之前的两个项目合并为一个，代码也得以通用。

项目经过 react 同构，之前的许多问题都得以解决：

1. 开发效率低的问题：同构应用只有一个项目和一套技术栈，只要拥有 react 开发经验，就可以快速投入前端和后端的开发当中；
2. 可维护性差的问题：同构应用可以进行大量的代码公用，包括工具方法、常量、页面组件和 redux 的大部分逻辑等，可重用性大大提高；
3. 首屏性能、SEO 等：有了服务端渲染，麻麻再也不担心首屏和 SEO 问题啦。

其实新的 react 服务端渲染架构并不是对之前『中途岛』的推翻，而是它理念的演进，核心思想都是由服务端来控制渲染，只是这里将之前互不干涉的前后端项目糅合到了一起，使用同构的方式简化了渲染层的工作。对于已使用 node 中间层的项目，不妨尝试一下 react 同构的技术方案，它会使你的开发效率和首屏性能得到飞速提升。