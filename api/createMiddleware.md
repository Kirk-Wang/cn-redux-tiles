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

`createMiddleware` take object as a parameter, and this object will be desctructured and it's properties will be available inside `fn` of the tile – so it works like [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection). It allows us to test our business logic really use, because we don't have to mock global imports, we just have to pass needed actions/selectors/etc.

While some might argue that it is actually anti-pattern, I personally think that it is a good strategy – less dependencies, more consistent access to other parts of state and dispatching other actions.

## waitTiles function and server-side rendering

Also, we get in returned object `waitTiles` function. What does it do? Well, this function gets active promises and waits for them – so if you dispatch 5 async modules, all of these promises will be collected by the middleware, and this function after invokation will wait for them. You can use it for server-side rendering – you do "dry-run" rendering first time, prefetching needed data in `componentWillMount`, then you await for `waitTiles()`, and after resolving render will give you markup with prefetched data!
Remember, though, that you have to instantiate new store for each request, otherwise this storage will be for requests, which will create a mess.
