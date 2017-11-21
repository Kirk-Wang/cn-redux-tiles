# CreateTile API

这是这个库的主要功能 - 它是一个工厂，它为您创建所有需要的实体：action，reducer，constant和selector。另外，“默认”`createTile`是异步的（我个人更经常地发现这样的模块，所以它是一个默认的版本），它支持这个开箱即用的生命周期。

基本的tile将如下所示：
```javascript
import { createTile } from 'redux-tiles';

const { action, selectors, reducer, constants, reflect } = createTile({
  type: ['user', 'info'],
  fn: ({ api, params }) => api.get('/user/info', params),
});
```

默认的数据结构，将数据保存在内部（如果使用`selectors.get`，将会返回）如下：
```javascript
{
  isPending: false, // true，当你的异步函数执行某些action
  fetched: true, // true，一旦结果被拉取，无论是成功还是失败
  data: { yourData: ... }, // 数据从promise中resolve出来，由你的函数返回
  error: null, // 将会是你的函数抛出的错误
}
```

你根本不能影响这个结构 – 你的数据将会转到`data`字段，或者在抛出错误的时候转到`error`。`fetched`字段允许你是否保存值在`data`内，所以你可以从`fn`返回而没有任何问题。通过这个契约你可以确定你组件里面的这个接口，所以没有 `isLoading`, `isPending`, `isUpdating`等等 – 对于每个用例，似乎都有更多的语义。是的，它不是那么优雅，但更一致;对于`data`字段使用相同参数。

## 用API来创建新的tile

现在让我们来看看当我们创建一个新的tile时传入的对象：

```javascript
import { createTile } from 'redux-tiles';

const userTile = createTile({
  // type将被反射在store，所以它将存储在state.hn_api.user下
  // （所以数组中的第一个元素可以像伞一样服务！）lalala,我不懂你的幽默~~
  type: ['hn_api', 'user'],
  
  // async tile的函数应该返回一个promise，否则会失败。
  // 函数获取一个参数，对象,它包含所有通过的中间件以及参数 – 具有已被调用的分发函数的对象
  // e.g. here: dispatch(actions.hn_api.user({ id: 'someID' }));
  // promise的结果将被放在state内部的`data`下
  fn: ({ api, params }) => api.get(`/api/user/${params.id}`),
  
  // nesting允许你分开你的数据（第一个参数是来自fn的params）。
  // 它的意思是所有请求将会有他们自己的isPending，fetched，data和error以及caching
  // 它是一个可选参数，支持任意嵌套
  nesting: ({ id }) => [id],

  // 这是一个对指定项目的主动缓存 – 
  // 如果它被下载，那么它将不会被再次下载，
  // 除非我们将调用第二个参数`forceAsync：true`：
  // dispatch(actions.hn_api.user({ id: 'someID' }, { forceAsync: true }));
  //
  // 同样，这意味着同时只有一个发出的请求，其他dispatch将返回完全相同的promise
  caching: true,
});
```

## 缓存

异步tile支持开箱即用的缓存，您只需将属性`caching`设置为`true`即可。这将会使两件事情发生 – 如果数据已经在那里显示，它将不会调用相同的函数，而且一旦它已经在处理中，它将防止再次调用相同的函数(并且dispatch过的action将返回完全相同的promise，所以你可以安全地等待它，然后查询状态 - 它将是一个更新过的值)。后一种情况很有趣 – 它基本上意味着我们摆脱了race condition，并且我们可以安全地以声明的方式查询相同的端点，而不必担心对相同端点的多个请求。

如果你已经dispatch了一个启用caching的action，并且你想要再次调用这个action，然后你将必须发送一个拥有key为`forceAsync: true` 的对象，来作为调用函数的第二个参数：
```js
dispatch(actions.api.users({ id: 'someID' }, { forceAsync: true }));
```

Though the same promise thing might seem as a magical part, it is not! In order to make it work for each requests in Node.js, we keep this object inside middleware (so it belongs to the instance of a store), and it means that in order to make it work we have to use [redux-tiles' middleware](./createMiddleware.md), or pass `promisesStorage` object to [redux-thunk](https://github.com/gaearon/redux-thunk):
```js
applyMiddleware(thunk.withExtraArgument({ promisesStorage: {} }))
```

Redux-tiles' middleware will inject this object automatically. This object is also crucial for server-side rendering, in case we want to prefetch data – this object collects all requests (regardless of caching; it will just filter same actions if caching is enabled) and `waitTiles` will await all of them.

## Function

Property `fn` is an obvious target to perform some API requests, but there are no restrictions on usage of it, so you can do whatever you want – starting from sleeping several seconds and to polling of some new data.

Let's implement both of these cases, starting with sleep. We will show notification for 5 seconds, and after close it:
```javascript
import { createTile } from 'redux-tiles';
import { sleep } from 'delounce';

const showNotification = createTile({
  type: ['ui', 'showNotifications'],
  fn: async ({ dispatch, actions, params }) => {
    dispatch(actions.ui.notifications(params));
    await sleep(5000);
    // let's assume that sending only id will close notification
    dispatch(actions.ui.notifications({ id: params.id }));

    // in this case this data is pretty much useless
    // but sometimes it is helpful
    return { closed: true };
  },
  // so we can check whether notification is going to be closed,
  // it is not really needed, but it costs nothing to us, so we can
  // remove/add it in no time
  nesting: ({ id }) => [id],
});
```

Now let's go to more complex example – polling of the user data. For example, we billed user, and now we are awaiting updated profile details:
```javascript
import { createTile } from 'redux-tiles';
import { sleep } from 'delounce';

const pollDetails = createTile({
  type: ['polling', 'details'],
  fn: async ({ dispatch, actions, params, selectors, getState }) => {
    while (true) {
      await sleep(3000);
      const { data } = await dispatch(actions.user.data());
      if (data.card !== params.card) {
        // same thing – this one is not particularly helpful data,
        // but it might be useful
        return { card: data.card };
      }
    }
  },
});
```

Polling is a great example of hitting the boundaries of this library – it is not intended to solve complex asynchronous flows, it is all about composition of small units. So, if you need even more complicated behaviour, or some tweeks like stopping of the polling, etc, then I can recommend to take a look at something like [Redux-saga](https://github.com/redux-saga/redux-saga).
But if you have them just in couple of places, when you should be good sticking with redux-tiles.

## Returned object API

When we create a new tile, we get in response object with all needed entities. As you will see later, it is advised to use special utilities to combine them together, but it is possible to do it by hand, in case you need. So, let's take a closer look:

```javascript
import { createTile } from 'redux-tiles';

const {
  // this is an actual action, which you can dispatch if you want to
  // there is no magic behind it, except that it has a property `reset`,
  // which is a function to reset reducer to default state (synchronously)
  action,

  // reducer function. this one is really tricky – in order to use it correctly,
  // you would have to combine it by yourself (so array passed in `type` will
  // make it not that easy)
  // also, in case you want to combine it by yourself, keep in mind that
  // it can be attached only to the root without using `createReducers`
  //
  // so, to conclude, this one, unfortunately, has some magic and it is better
  // to combine using `createReducers` from this library
  reducer,

  // object with START, SUCCESS, FAILURE and RESET strings
  // strings are unique by using type
  // you can use them anywhere you want – for instance, in redux-saga
  constants,

  // object with two methods – get and getAll
  // get can get nested data, while getAll returns you all data
  // despite of nesting. getAll is a very rare function, you
  // will be good with just get in 99%
  selectors,

  // passed `type` property. can be also reached via `reflect` property
  tileName,

  // object which you passed to `createTile`. that's the way to get
  // your functions and to test them
  reflect
} = createTile({
  type: ['api', 'get', 'client'],
  fn: ({ api }) => api.get('/client/'),
});
```

In fact, you likely won't need to get any of those data from a tile, because there are utilities to combine all of them together, but it is helpful for understanding, that under the hood all components are presented.

## More information

* [Nesting](../advanced/nesting.md)
* [Selectors](../advanced/selectors.md)
