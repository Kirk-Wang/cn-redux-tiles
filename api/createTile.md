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

虽然相同的promise可能看起来像一个神奇的部分，但它不是！为了使它适用于Node.js中的每个请求，我们把这个对象保存在中间件中（所以它属于一个store的实例），这意味着为了使它能够工作，我们必须使用[redux-tiles的中间件](./createMiddleware.md)，或者把`promisesStorage`对象传递给[redux-thunk](https://github.com/gaearon/redux-thunk)：
```js
applyMiddleware(thunk.withExtraArgument({ promisesStorage: {} }))
```

Redux-Tile的中间件会自动注入这个对象。如果我们要prefetch数据，这个对象对于服务器端渲染也很重要 – 这个对象收集所有请求（不管是否缓存;如果启用了缓存，它只会过滤相同的action），并且`waitTiles` 将等待所有的请求。

## Function

属性`fn`是执行一些API请求的明显目标，但是对它的使用没有任何限制，所以你可以做任何你想做的事情 - 从睡眠几秒钟和轮询一些新的数据开始。

从睡眠开始，我们来实现这两种情况。我们将显示通知5秒钟后关闭它：
```javascript
import { createTile } from 'redux-tiles';
import { sleep } from 'delounce';

const showNotification = createTile({
  type: ['ui', 'showNotifications'],
  fn: async ({ dispatch, actions, params }) => {
    dispatch(actions.ui.notifications(params));
    await sleep(5000);
    // 让我们假设只发送id将关闭通知
    dispatch(actions.ui.notifications({ id: params.id }));

    //在这种情况下，这个数据几乎是无用的
    //但有时它是有帮助的
    return { closed: true };
  },
  // 所以我们可以检查通知是否将被关闭，
  // 它并不是真的需要，但它对我们没有任何成本，所以我们可以立即删除/添加它
  nesting: ({ id }) => [id],
});
```

现在我们来看更复杂的例子 - 轮询用户数据。 例如，我们为用户付款，现在我们正在等待更新的配置文件详细信息：
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
        // 同样的事情——这个数据不是特别有用，但可能有用
        return { card: data.card };
      }
    }
  },
});
```

轮询是触及这个库边界的一个很好的例子 – 它不是用来解决复杂的异步流，而是关于小单元的组合。因此，如果你需要更复杂的行为，或者像停止轮询之类的琐事，那么我可以建议你看一下[Redux-saga](https://github.com/redux-saga/redux-saga)之类的东西。
但如果你只在几个地方有他们，当你应该坚持使用redux-tiles的时候。

## Returned object API

当我们创建一个新的tile时，我们得到所有需要的实体的响应对象。后面你会看到，建议使用特殊工具将它们组合在一起，但是可以用手工完成，以防万一需要。 所以，让我们仔细看看：

```javascript
import { createTile } from 'redux-tiles';

const {
  // 这是一个实际的动作，如果你愿意，你可以dispatch
  // 它背后没有魔法，除了它有一个`reset`属性，
  // 这是一个将reducer重置为默认状态（同步）的函数
  action,

  // reducer function. 这个很棘手，为了正确地使用它，
  // 你将不得不自己结合它（所以在`type`中传入的数组将不会那么容易）
  // 另外，如果你想自己结合它，请记住，它只能在不使用`createReducers`的前提下附加到root上。
  // 所以，总之，不幸的是，这一个有一些魔力，最好使用这个库中的`createReducers`来合并
  reducer,

  // 有START，SUCCESS，FAILURE和RESET字符串的对象
  // 字符串通过使用type是唯一的
  // 你可以在任何你想要的地方使用它们 - 例如，在redox-saga中
  constants,

  // 有两个方法的对象 - get和getAll
  // get可以得到嵌套的数据，而getAll返回所有的数据，尽管它们都是嵌套的。
  // getAll是一个不常用的函数，99%的情况下，你仅用get就够了
  selectors,

  // 传递的`type`属性。 也可以通过`reflect`属性拿到
  tileName,

  // 你传递给`createTile`的对象。这就是得到函数和测试它们的方法
  reflect
} = createTile({
  type: ['api', 'get', 'client'],
  fn: ({ api }) => api.get('/client/'),
});
```

实际上，您可能不需要从tile中获得这些数据，因为有实用程序将它们组合在一起，但它有助于理解，在引擎盖下所有组件都被呈现出来了。

## 更多的信息

* [Nesting](../advanced/nesting.md)
* [Selectors](../advanced/selectors.md)
