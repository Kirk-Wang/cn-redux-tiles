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

这个软件包是基于我心中的这个想法构建的，人们通常会使用一些打包工具 - [Webpack](https://webpack.js.org/), [Browserify](http://browserify.org/) or [Rollup](http://rollupjs.org/)。 该包本身是用TypeScript编写的，并且因此可以提供立即可用的typings。

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
  // 它使我们能够更容易地进行测试，并且也可以组成其他的tiles
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

有足够的项目来保持您的状态管理清晰([例如](https://github.com/erikras/ducks-modular-redux))，但是它们主要是关于组织的，而不是从开发人员那里移除重复内容的麻烦。其他软件包为您提供了与REST-API的全面集成，[规范](https://github.com/paularmstrong/normalizr)你的实体，建立模型之间的关系等等。这里没有这个东西 – 事实上，如果你需要类似这样的东西，并且有能力查询你的本地“数据库”，我强烈建议你创建你自己的解决方案，它将根据你的具体问题定制。

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

// 我们将`actions`和`selectors`注入到中间件中，
// 所以它们将在所有tiles的`fn`函数内可用
const { middleware } = createMiddleware({ actions, selectors });

createStore(reducer, applyMiddleware(middleware));
```

## Tiles API

Tiles是这个库的核心。他们的目的是要非常容易组合，撰写和测试。有两种类型的tiles - 异步和同步。现代应用程序是非常动态的，所以异步的tile可能会更频繁地使用。另外，不要将自己限制在异步tiles仅用于API通信的思维模式中 – 它可能是任何事情，涉及一些异步交互（以及组成其他tiles） - 例如，实现长轮询。

> [完整的异步tiles文档](https://redux-tiles.js.org/api/createTile.html)

```javascript
import { createTile } from 'redux-tiles';

const photos = createTile({
  // 它们将在redux状态下被构造成api.photos，
  // 也可以在actions和selectors下使用：
  // actions.tiles.api.photos
  type: ['api', 'photos'],
  
  
  // params是我们派发动作的一个对象
  // 你只能传递一个参数，所以保持它作为一个对象
  // 具有不同的属性
  //
  // 所有其他属性都来自您的中间件
  // fn期望从这个函数中获得Promise
  fn: ({ params, api }) => api.get('/photos', params),
  
  
  // 嵌套数据：
  // { 5:
  //    10: {
  //      isPending: true,
  //      fetched: false,
  //      data: null,
  //      error: null,
  //   },
  // },
  // 如果保存在同一个嵌套数组下，数据将被替换
  // 其他分支将被合并
  nesting: (params) => [params.page, params.count],

  // 除非我们用asyncForce: true作为第二个参数对象来调用，
  // 否则它将不会再请求
  // dispatch(actions.tiles.api.photos(params, { asyncForce: true }))

  caching: true,
});
```

我们有时还想保留一些同步信息（例如通知列表），或者我们想为计算器存储一些数字，或者激活过滤器（[todoMVC](http://todomvc.com/)是很多同步操作的一个很好的例子）。在这种情况下，我们将使用`createSyncTile`，它没有像`isPending`，`error`或`fetched`这样的元数据，而是直接将所有从函数返回的数据保存在状态中。

> [完整的同步tiles文档](https://redux-tiles.js.org/api/createSyncTile.html)

```javascript
import { createSyncTile } from 'redux-tiles';

const notifications = createSyncTile({
  type: ['notifications'],
  
  // 所有参数与异步tile相同
  fn: ({ params, dispatch, actions }) => {
    // 我们可以派遣异步actions - 但我们不能等待它返回结果
    // 在同步tiles里面
    dispatch(actions.tiles.user.dismissTerms());

    return {
      type: params.type,
      data: processData(params.data),
    };
  },

  // 或者，如果您对现有数据执行某些actions，
  // 编写更多的声明性actions可能是有用的
  // 它们具有完全相同的签名并分派返回的数据
  // 到tile
  fns: {
    add: ({ params, selectors, getState}) => {
      const currentData = selectors.notifications(getState(), params);

      return {
        ...currentData,
        data: currentData.concat(params.data),
      };
    },
  },

  // 你可以通过初始状态来同步tile
  // 请小心！ 如果你使用嵌套，那么
  // 你必须指定嵌套的项目（否则selectors会
  // 从您的嵌套项目中返回undefined）
  initialState: {
    terms: {
      type: 'terms',
      data: []
    },
  },
  
  // 嵌套以同样的方式工作
  nesting: ({ type }) => [type],
});
```

## 嵌套

> [关于嵌套的完整文档](https://redux-tiles.js.org/advanced/nesting.html)

我们经常需要分离一些信息，并且使用规范的redux，我们必须这样写：

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

或者用`Object.assign`，这会使得它更不可读。这是一个很常见的模式，也很容易出错 – 所以我们必须用单元测试来覆盖这样的代码，而实际上他们并没有做太多的内在逻辑 – 只是合并。当然，我们可以使用像`lodash.merge`这样的东西，但并不总是合适的。在tiles中我们有`nesting`属性，您可以在其中指定一个函数，您可以从中返回一个嵌套值数组。与上面相同的代码：

```javascript
const infoTile = createTile({
  type: ['info', 'storage'],
  
  // 这里的参数和嵌套是同一个对象
  fn: ({ params: { quantity, id }, api }) => api.get('/storage', { quantity, id }),
  
  // 在这种状态下，他们将保持以下结构：
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

## 中间件

为了使用这个库，你必须应用中间件，它将处理从分发actions返回的函数。这个包提供了非常基本的一个：

> [完整的中间件文档](https://redux-tiles.js.org/api/createMiddleware.html)

```javascript
import { createMiddleware } from 'redux-tiles';

// 这些不是必需的，但添加它们可以让你
// 做依赖注入模式，所以测试起来比较容易
import actions from '../actions';
import selectors from '../selectors';


// 将API层放入中间件是一个好主意，所以
// 例如，您可以轻松分离客户端和服务器
import api from '../utils/api';


// 这个对象是可选的。 每个属性将在所有tiles的`fn`里面可用
// 同样，`waitTiles`对于服务器端的渲染是有帮助的
const { middleware, waitTiles } = createMiddleware({ actions, selectors, api });
applyMiddleware(middleware);
```

此外，支持[redux-thunk]（https://github.com/gaearon/redux-thunk），为了传递你自己的属性，你应该[注入这个对象到redux-thunk](https://github.com/gaearon/redux-thunk#injecting-a-custom-argument)。 而且，在文件之上导入actions和selectors并没有什么不好，但是测试可能需要更多的模拟，这会使测试变得更加脆弱。

## 服务器端渲染

> [有关SSR与prefetch的文章](http://blog.bloomca.me/2017/06/11/server-side-rendering-with-prefetch.html)

Redux-tiles支持服务器端的请求。为了正确地做到这一点，您应该在Node.js中为每个请求创建操作。Redux-Tiles缓存异步请求（并将它们保存在中间件中，所以它们不会在不同的用户请求之间共享） – 它保留所有活动promise的列表，所以你可能无意中与其他用户分享这部分内存！

另外，为了使这部分功能正常工作，必须使用redux-tiles中间件，或将`promisesStorage`对象传递给redux-thunk附加对象（更多内容请参见[文档中的缓存部分](https://redux-tiles.js.org/api/createTile.html#caching)）。

```javascript
import { createMiddleware, createEntities } from 'redux-tiles';
import { createStore, applyMiddleware } from 'redux';
import tiles from '../../common/tiles';

const { actions, reducer, selectors } = createEntities(tiles);
const { middleware, waitTiles } = createMiddleware({ actions, selectors });
const store = createStore(reducer, {}, applyMiddleware(middleware));


// 这是一个徒劳的渲染 只需要启动请求
// 不幸的是，没有办法避免它
renderApplication(req);

// 等待渲染过程中触发的所有请求
await waitTiles();


// 这次您可以安全地呈现您的应用程序 - `componentWillMount`中的所有请求都将全部填满
// 记住，在服务器上不会触发`componentDidMount`
res.send(renderApplication(req));
```

还有一个包[delounce](https://github.com/Bloomca/delounce)，你可以从那里获得`limit`函数，如果请求时间太长，这个函数将会渲染应用程序。

## 选择器

> [关于选择器的完整文档](https://redux-tiles.js.org/advanced/selectors.html)

所有的tiles都提供selectors。 在收集完所有的tiles之后，调用`createSelectors`函数，可能会更改默认的名称空间，然后可以根据传入的类型使用它：

```javascript
import { createTile, createSelectors } from 'redux-tiles';

const tile = createTile({
  type: ['user', 'auth'],
  fn: ...,
  nesting: ({ id }) => [id],
});

const tiles = [tile];

const selectors = createSelectors(tiles);

// 第二个参数是你分发action的参数 - 它将得到相应嵌套的数据
const { isPending, fetched, data, error } = selectors.user.auth(state, { id: '456' });
```

## 测试

几乎所有的业务逻辑将被包含在“复杂”的tiles中，而这些tiles不会自己发出请求，而是调度其他tiles，从而组成结果。通过中间件传递所有需要的功能是非常重要的，所以你可以很容易地模拟它，而不依赖于其他模块。所有传递的数据都可以通过在tiles中的`reflect`属性获得。

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
