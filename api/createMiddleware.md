# 中间件概述

[中间件](http://redux.js.org/docs/advanced/Middleware.html)在Redux中处理action，所以你不仅可以返回带有`type`和`payload`属性的普通对象，还可以返回promise和函数等等 – 只要用适当的中间件来处理。有几个非常受欢迎的库，并且最基本的和流行的，我认为是[redux-thunk](https://github.com/gaearon/redux-thunk)，处理异步请求的最基本的中间件。虽然这是一个非常棒的中间件，但它却很简单，我建议使用这个库中的中间件（我也强烈建议看一下它们的源代码 – 它们非常简单 - [redux-tiles 中间件](https://github.com/Bloomca/redux-tiles/blob/master/src/middleware.ts)和[redux-thunk 中间件](https://github.com/gaearon/redux-thunk/blob/master/src/index.js)）。

## createMiddleware API

`createMiddleware` 做两件事 – 创建实际的中间件，并且还暴露了函数去“等待”所有分发了的promise。 我们来看一下这个例子：

```javascript
import { createMiddleware } from 'redux-tiles';
import { createStore, applyMiddleware } from 'redux';
import api from '../api';
import { actions, selectors, reducer } from '../entities';

const { middleware, waitTiles } = createMiddleware({ api, actions, selectors });

const store = createStore(reducer, applyMiddleware(middleware));
```

`createMiddleware`将对象作为参数，并且此对象将被解析并且它的属性将在tile的fn内可用 – 所以它像[dependency injection](https://en.wikipedia.org/wiki/Dependency_injection)一样工作。它使我们能够测试我们真正使用的业务逻辑，因为我们不必模拟全局import，只需通过所需的actions/selectors/等等。

虽然有些人可能会认为它实际上是反模式，但我个人认为这是一个很好的策略 – 减少依赖性，更一致地访问state的其他部分，并分发其他action。

## waitTiles函数和服务器端渲染

另外，我们得到了返回对象的`waitTiles`函数。 它有什么作用？ 这个函数会得到所有活动的promise并等待它们 – 所以如果你分发了5个异步模块，所有这些promise都会被中间件收集，调用后这个函数会等待它们。您可以将其用于服务器端呈现 – 你首先进行“dry-run”渲染，在`componentWillMount`中预取所需的数据，然后等待`waitTiles()`，解析渲染后会产生给你带预取数据的标记！但请记住，您必须为每个请求实例化新的store，否则这个存储将用于请求，这会造成混乱。
