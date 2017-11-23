# 嵌套

很多时候我们有一些实体应该以同样的方式处理（例如，相同的缓存，相同的请求，在存储之前对它们进行相同的操作），但是被一些标识符（或者可能是几个）分开。规范的例子是通过id – 逐个下载项目; 通常我们有一个特定的项目的页面，我们下载，然后存储在state，通过id分隔。

这就是在带有[redux-thunk中间件](https://github.com/gaearon/redux-thunk)传统的redux中看起来的样子 – 我们从action开始吧：
```js
import api from '../api';
export const FETCH_ITEM_START = 'FETCH_ITEM_START';
export const FETCH_ITEM_FAILURE = 'FETCH_ITEM_FAILURE';
export const FETCH_ITEM_SUCCESS = 'FETCH_ITEM_SUCCESS';

export const fetchItem = ({ id }) => {
  return (dispatch) => {
    dispatch({
      type: FETCH_ITEM_START,
      payload: { id },
    });

    return api
      .get(`/api/item/${id}`)
      .then(data => dispatch({
        type: FETCH_ITEM_SUCCESS,
        payload: { data, id },
      }))
      .catch(error => dispatch({
        type: FETCH_ITEM_FAILURE,
        payload: { id },
        error,
      }));
  };
};
```

你有没有注意到每次我们都在每个dispatch调用中传递了`payload: { id }`？这是因为我们必须区分每个请求（所以我们知道哪个请求正在加载，哪个已经完成）！因此，在reducer中，我们必须谨慎地合并我们的状态：

```js
export const reducer = (state, action) => {
  switch (action.type) {
    case FETCH_ITEM_START:
      return {
        ...state,
        [action.payload.id]: {
          isPending: true,
          fetched: false,
          error: null,
          data: null,
        },
      };
    case FETCH_ITEM_FAILURE:
      return {
        ...state,
        [action.payload.id]: {
          isPending: false,
          // 注意，尽管有错误，但是fetched是true！
          fetched: true,
          error: action.error,
          data: null,
        },
      };
    case FETCH_ITEM_SUCCESS:
      return {
        ...state,
        [actions.payload.id]: {
          isPending: false,
          fetched: true,
          error: null,
          data: action.payload.data,
        },
      };
    default:
      return state;
  }
}
```

所以，我们必须手工合并所有的东西，因此会引入一些问题。首先，我们违反了[DRY principle](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)，其次，我们为bug增加了一个可能的位置，所以我们必须为这个功能编写测试，第三，它是冗长的。

Redux-tiles旨在用一行代码解决这个确切的问题。 相同的确切代码将使用tile，看起来如下：

```js
import { createTile } from 'redux-tiles';

export const itemTile = createTile({
  type: ['api', 'item'],
  fn: ({ api, params }) => api.get(`/api/item/${params.id}`),
  // 我们应该返回一个数组，它可以有任何长度，并将被正确合并到整个深度
  nesting: ({ id }) => [id],
  // 作为一个免费的奖励，我们可以缓存我们的响应，并且他们也是通过`id`值被缓存的
  caching: true,
});
```

这段代码和“原始”redux代码中所描述的完全一样 – 它将获取给定的资源，并将所有元数据置于state中的`id`之下。

我们来看另一个例子，分页。 通常分页我们指定两个东西－page size和page number。 所以让我们编写这个tile：

```js
const itemsWithPaginationTile = createTile({
  type: ['api', 'items'],
  fn: ({ params, api }) => api.get('/api/items', {
    pageSize: params.pageSize,
    pageNumber: params.pageNumber,
  }),
  // 我们在这里有更深的嵌套 – 它将被正确处理！
  nesting: (params) => [params.pageSize, params.pageNumber],
  // 页面将被独立缓存
  caching: true,
});
```

此外，它的工作方式与同步tile完全相同，所以如果您想通过某些标识符分隔的保留某些同步数据，则完全适合您。

所以，正如你所看到的，redux-tile减轻了你的一些负担 – 你不必测试你的合并实现，并且你可以很容易地获得嵌套项的缓存。