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
  
  // function for async tile should return a promise, otherwise it
  // will fail. function gets one parameter, object, which contains
  // all passed middleware, and params – object with which dispatched
  // function was invoked
  // e.g. here: dispatch(actions.hn_api.user({ id: 'someID' }));
  // result of the promise will be placed under `data` inside state
  // async tile的函数应该返回一个promise，否则会失败。
  // 函数获取一个参数，一个包含所有通过的中间件以及参数的对象 – 
  // 
  fn: ({ api, params }) => api.get(`/api/user/${params.id}`),
  
  // nesting allows you to separate your data (first argument is params
  // from fn). It means that all requests will have their own isPending,
  // fetched, data and error, as well as caching
  // it is an optional parameter, arbitrary nesting is supported
  nesting: ({ id }) => [id],

  // this is an aggressive caching for specific item – if it was
  // downloaded, then it won't be downloaded again at all, unless
  // we will invoke with the second parameter `forceAsync: true`:
  // dispatch(actions.hn_api.user({ id: 'someID' }, { forceAsync: true }));
  //
  // also, it means that there will be only one simulatenous request
  // other dispatches will return exactly the same promise
  caching: true,
});
```

## Caching

Asynchronous tiles support caching out of the box, you just have to set property `caching` to `true`. It will make two things happen – it won't invoke the same function if the data is already presented there, and also it will prevent the same function be invoked again in case it is already being processing (but dispatched action will return exactly the same promise, so you can safely await for it, and then query the state - it will be an updated value). The latter case is interesting – it basically means that we get rid of race conditions, and we are safe to query same endpoints in a declarative way, without worrying of several requests to same endpoints.

If you have already dispatched an action with enabled caching, and you want to invoke this action again, then you would have to send an object with a key `forceAsync: true` as a second parameter to invoked function:
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
