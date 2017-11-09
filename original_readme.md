# Redux-Tiles

[![Build Status](https://travis-ci.org/Bloomca/redux-tiles.svg?branch=master)](https://travis-ci.org/Bloomca/redux-tiles)
[![npm version](https://badge.fury.io/js/redux-tiles.svg)](https://badge.fury.io/js/redux-tiles)
[![Coverage Status](https://coveralls.io/repos/github/Bloomca/redux-tiles/badge.svg?branch=master)](https://coveralls.io/github/Bloomca/redux-tiles?branch=master)
[![dependencies Status](https://david-dm.org/bloomca/redux-tiles/status.svg)](https://david-dm.org/bloomca/redux-tiles)

**[英文文档](https://redux-tiles.js.org/)**

Redux是一个非常好的库，可以让状态管理变得清晰。但问题在于它的样板代码写的太冗长了，而且你经常会觉得你一次又一次地做同样的事情。 这个库试图在Redux之上提供最小的抽象，以允许轻松的组合、容易的异步请求和健全的可测试性。可以在[现有项目中](https://bloomca.github.io/redux-tiles/advanced/integration.html)开始使用此库, 并逐步添加新功能。

>**[更多关于这个库背后的基本原理](http://blog.bloomca.me/2017/06/02/why-i-created-redux-tiles-library.html)**<br>
>
>**[实例](./examples)**
> * [Calculator](./examples/calculator)
> * [TodoMVC](./examples/todomvc)
> * [Hacker News API](./examples/hacker-news-api)
> * [Github API](./examples/github-api)

## 安装

要安装最新的稳定版本，请运行：
```shell
npm install --save redux-tiles
```

这个软件包的构思是基于我的这个想法，人们通常会使用一些打包工具 - [Webpack](https://webpack.js.org/), [Browserify](http://browserify.org/) or [Rollup](http://rollupjs.org/)。 该包本身是用TypeScript编写的，并且因此可以提供立即可用的typings。

如果由于某种原因不使用bundler，则可以使用位于[dist文件夹](https://unpkg.com/redux-tiles@0.6.1/dist/)中的UMD版本。 只需在你的页面中通过`script`标签包含它，然后你就可以在window.ReduxTiles全局变量下得到它。

## TOC:

- [使用示例](#user-content-example-of-use)
- [基本原理](#user-content-rationale)
- [集成API](#user-content-integration-api)
- [Tiles API](#user-content-tiles-api)
- [嵌套](#user-content-nesting)
- [中间件](#user-content-middleware)
- [服务端渲染](#user-content-server-side-rendering)
- [选择器](#user-content-selectors)
- [测试](#user-content-tests)

## 使用示例

> [更全面的例子](https://redux-tiles.js.org/introduction/Example.html)

```javascript
import { createTile, createSyncTile } from 'redux-tiles';

// 同步tile存储信息没有任何异步的东西
const loginStatus = createSyncTile({
  type: ['user', 'loginStatus'],
  fn: ({ params: status }) => ({
    status,
    timestamp: Date.now(),
  }),
});

// 请求到服务器
// 它是绝对分开的，所以很容易
// 组合不同的请求
const authRequest = createTile({
  type: ['user', 'authRequest'],
  // 我们可以访问dispatch, actions, selectors, 等 -
  // 我们可以在创建中间件时传递所需的所有内容
  // 它使我们能够更容易地进行测试，也可以组成其他的tiles
  fn: ({ params, api, dispatch, actions, getState }) =>
    api.post('/login', params),
});

// 实际的业务逻辑
// 请注意，我们不在这里使用直接的`api`调用
// 我们只是组成其他基本的tiles
const authUser = createTile({
  type: ['user', 'auth'],
  fn: async ({ params, dispatch, actions, selectors, getState }) => {
    // 登录用户
    const { data: { id }, error } = await dispatch(actions.tiles.user.authRequest(params));

    if (error) {
      throw new Error(error);
    }

    // 同步地设置用户状态
    dispatch(actions.tiles.user.loginStatus(true));

    return true;
  },
});
```

## 基本原理

有足够的项目来保持您的状态管理清晰([例如](https://github.com/erikras/ducks-modular-redux))，但是它们主要是关于组织的，而不是从开发人员那里删除重复内容的麻烦。其他软件包为您提供了与REST-API的全面集成，[规范](https://github.com/paularmstrong/normalizr)你的实体，建立模型之间的关系等等。这里没有这个东西 – 事实上，如果你需要类似这样的东西，并且有能力查询你的本地“数据库”，我强烈建议你创建你自己的解决方案，它将根据你的具体问题定制。

这个软件包专注于非常基本的模块，这对于非常简单的应用来说是很好的(e.g. login/logout, fetch client data, set up calculator values).

## 集成API

尽管使用易于使用的软件包编写新模块，但您还是必须做一些工作才能将其集成到您的项目中。 简而言之，你必须有一个中间件来处理派遣动作的返回函数（在这个包中提供了一个，并且[redux-thunk](https://github.com/gaearon/redux-thunk)也可以），然后你必须结合所有的模块来创建actions和reducers。 在这个小例子中我们可以看到：

```javascript
import { createTile, createEntities, createMiddleware } from 'redux-tiles';
import { createStore, applyMiddleware } from 'redux';

const clientDataTile = createTile({
  type: ['client', 'data'],
  fn: ({ api, params}) => api.get('/client/info'),
});

const tiles = [
  clientDataTile,
];

const { actions, reducer, selectors } = createEntities(tiles);

// we inject `actions` and `selectors` into middleware, so they
// will be available inside `fn` function of all tiles
const { middleware } = createMiddleware({ actions, selectors });

createStore(reducer, applyMiddleware(middleware));
```

## Tiles API

Tiles are the heart of this library. They are intended to be very easy to use, compose and to test.
There are two types of tiles – asynchronous and synchronous. Modern applications are very dynamic, so async ones will be likely used more often. Also, don't constrain yourself into the mindset that async tiles are only for API communication – it might be anything, which involves some asynchronous interaction (as well as composing other tiles) – for instance, long polling implementation.
> [Full documentation for async tiles](https://redux-tiles.js.org/api/createTile.html)
```javascript
import { createTile } from 'redux-tiles';

const photos = createTile({
  // they will be structured api.photos inside redux state,
  // and also available under actions and selectors as:
  // actions.tiles.api.photos
  type: ['api', 'photos'],
  
  
  // params is an object with which we dispatch the action
  // you can pass only one parameter, so keep it as an object
  // with different properties
  //
  // all other properties are from your middleware
  // fn expects promise out of this function
  fn: ({ params, api }) => api.get('/photos', params),
  
  
  // to nest data:
  // { 5:
  //    10: {
  //      isPending: true,
  //      fetched: false,
  //      data: null,
  //      error: null,
  //   },
  // },
  // if you save under the same nesting array, data will be replaced
  // other branches will be merged
  nesting: (params) => [params.page, params.count],


  // unless we will invoke with second parameter object with asyncForce: true,
  // it won't be requested again
  // dispatch(actions.tiles.api.photos(params, { asyncForce: true }))
  caching: true,
});
```

We also sometimes want to keep some sync info (e.g. list of notifications), or we want to store some numbers for calculator, or active filters ([todoMVC](http://todomvc.com/) is a good example of a lot of synchronous operations). In this situation we will use `createSyncTile`, which has no meta data like `isPending`, `error` or `fetched`, but keeps all returned data from a function directly in state.

> [Full documentation for sync tiles](https://redux-tiles.js.org/api/createSyncTile.html)

```javascript
import { createSyncTile } from 'redux-tiles';

const notifications = createSyncTile({
  type: ['notifications'],
  
  
  // all parameters are the same as in async tile
  fn: ({ params, dispatch, actions }) => {
    // we can dispatch async actions – but we can't wait
    // for it inside sync tiles
    dispatch(actions.tiles.user.dismissTerms());

    return {
      type: params.type,
      data: processData(params.data),
    };
  },

  // alternatively, if you perform some actions on existing data,
  // it might be useful to write more declarative actions
  // they have exactly the same signature and dispatch returned data
  // to the tile
  fns: {
    add: ({ params, selectors, getState}) => {
      const currentData = selectors.notifications(getState(), params);

      return {
        ...currentData,
        data: currentData.concat(params.data),
      };
    },
  },

  // you can pass initial state to sync tile
  // please, be careful with it! if you use nesting, then
  // you have to specify nested items (otherwise selectors will
  // return undefined for your nested item)
  initialState: {
    terms: {
      type: 'terms',
      data: []
    },
  },
  
  // nesting works the same way
  nesting: ({ type }) => [type],
});
```

## Nesting

> [Full documentation on nesting](https://redux-tiles.js.org/advanced/nesting.html)

Very often we have to separate some info, and with canonical redux we have to write something like this:
```javascript
case ACTION.SOME_CONSTANT:
  return {
    ...state,
    [action.payload.id]: {
      [action.payload.quantity]: {
        ...state[action.payload.id],
        ...action.payload.data,
      },
    },
  };
```

Or with `Object.assign`, which will make it even less readable. This is a pretty common pattern, and also pretty error prone – so we have to cover such code with unit-tests, while in reality they don't do a lot of intrinsic logic – just merge. Of course, we can use something like `lodash.merge`, but it is not always suitable. In tiles we have `nesting` property, in which you can specify a function from which you can return an array of nested values. The same code as above:

```javascript
const infoTile = createTile({
  type: ['info', 'storage'],
  
  // params here and in nesting are the same object
  fn: ({ params: { quantity, id }, api }) => api.get('/storage', { quantity, id }),
  
  // in the state they will be kept with the following structure:
  // {
  //   someId: {
  //     5: {
  //       isPending: true,
  //       fetched: false,
  //       data: null,
  //       error: null,
  //     },
  //   },
  // }
  nesting: ({ quantity, id }) => [id, quantity],
});
```

## Middleware

In order to use this library, you have to apply middleware, which will handle functions returned from dispatched actions. Very basic one is provided by this package:

> [Full documentation for middleware](https://redux-tiles.js.org/api/createMiddleware.html)
```javascript
import { createMiddleware } from 'redux-tiles';

// these are not required, but adding them allows you
// to do Dependency Injection pattern, so it is easier to test
import actions from '../actions';
import selectors from '../selectors';

// it is a good idea to put API layer inside middleware, so
// you can easily separate client and server, for instance
import api from '../utils/api';


// this object is optional. every property will be available inside
// `fn` of all tiles
// also, `waitTiles` is helpful for server-side-rendering
const { middleware, waitTiles } = createMiddleware({ actions, selectors, api });
applyMiddleware(middleware);
```

Also, [redux-thunk](https://github.com/gaearon/redux-thunk) is supported, and in order to pass your own properties you should [inject this object to redux-thunk](https://github.com/gaearon/redux-thunk#injecting-a-custom-argument). Also, there is nothing bad to just import actions and selectors on top of the files, but then testing might require much more mocking, which can make your tests more brittle.

## Server-side Rendering

> [Article about SSR with prefetch](http://blog.bloomca.me/2017/06/11/server-side-rendering-with-prefetch.html)

Redux-tiles support requests on the server side. In order to do that correctly, you are supposed to create actions for each request in Node.js. Redux-Tiles has caching for async requests (and keeps them inside middleware, so they are not shared between different user requests) – it keeps list of all active promises, so you might accidentaly share this part of the memory with other users!

Also, to make this part of functionality working, you have to use redux-tiles middleware, or pass `promisesStorage` object to redux-thunk additional object (more in [caching section in docs](https://redux-tiles.js.org/api/createTile.html#caching)).

```javascript
import { createMiddleware, createEntities } from 'redux-tiles';
import { createStore, applyMiddleware } from 'redux';
import tiles from '../../common/tiles';

const { actions, reducer, selectors } = createEntities(tiles);
const { middleware, waitTiles } = createMiddleware({ actions, selectors });
const store = createStore(reducer, {}, applyMiddleware(middleware));

// this is a futile render. It is needed only to kickstart requests
// unfortunately, there is no way to avoid it
renderApplication(req);

// wait for all requests which were fired during the render
await waitTiles();

// this time you can safely render your application – all requests
// which were in `componentWillMount` will be fullfilled
// remember, `componentDidMount` is not fired on the server
res.send(renderApplication(req));
```

There is also a package [delounce](https://github.com/Bloomca/delounce), from where you can get `limit` function, which will render the application if requests are taking too long.

## Selectors

> [Full documentation on selectors](https://redux-tiles.js.org/advanced/selectors.html)

All tiles provide selectors. After you've collected all tiles, invoke `createSelectors` function with possible change of default namespace, and after you can just use it based on the passed type:

```javascript
import { createTile, createSelectors } from 'redux-tiles';

const tile = createTile({
  type: ['user', 'auth'],
  fn: ...,
  nesting: ({ id }) => [id],
});

const tiles = [tile];

const selectors = createSelectors(tiles);

// second argument is params with which you dispatch action – it will get data
// for corresponding nesting
const { isPending, fetched, data, error } = selectors.user.auth(state, { id: '456' });
```

## Tests

Almost all business logic will be contained in "complex" tiles, which don't do requests by themselves, rather dispatching other tiles, composing results from them. It is very important to pass all needed functions via middleware, so you can easily mock it without relying on other modules. All passed data is available in tiles via `reflect` property.

```javascript
import { createTile } from 'redux-tiles';

const params = {
  type: ['auth', 'token'],
  fn: ({ api, params }) => api.post('/token', params),
};
const tile = createTile(params);

// same object
assert(tile.reflect === params); // true
```

## Contributing

All suggestions or participating are welcome! If you have any idea about improving API, or bringing some common functionality, don't hesitate, but please create an issue.
Also, in case you really think something is missing or wrong, please create an issue first, where we will discuss the problem and possible solutions, and then we can agree on implementation details.

## LICENSE

MIT
