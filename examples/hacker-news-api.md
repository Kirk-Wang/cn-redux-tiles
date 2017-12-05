# HackerNews API 与 redux-tiles

让我们为hacker news构建一个真实的API tiles。我们将创建一个简单的功能，它允许我们按类别下载页面(例如:top stories，或new stories)，并自动下载和填充条目。我们将使用官方的[HN API](https://github.com/HackerNews/API)，这是在firebase上托管的。API设置超出了本教程的范围，所以如果您有更多的问题，请使用以下链接：
- [Implementation in this example](https://github.com/Bloomca/redux-tiles/blob/master/examples/hacker-news-api/api.js)
- [Firebase](https://firebase.google.com/)
- [Web setup](https://firebase.google.com/docs/web/setup)

我们将开始按类型下载stories列表。 Hackernews API只返回一个大小为500的数组，我们将在稍后添加分页：

```javascript
export const storiesTile = createTile({
  type: ['hn_api', 'stories'],
  fn: ({ api, params }) => api.child(params.type).once('value').then(snapshot => snapshot.val()),
  nesting: ({ type }) => [type],
  caching: true,
});
```

尽管使用了Firebase，但在这里我们并不想实时更新，仅为了简单起见，所以我们不订阅或者做类似的事情。我们可以得到500个给定类型的条目 - 所以我们可以使用这个tile下载`topstories`，`new`和其他类别。我们也缓存结果，所以每次我们只需要一个页面就可以声明地调用它，并且确保它不会被再次下载。

现在让我们为单个项目创建tile。它将只下载一个条目，将其置于id命名空间下:

```javascript
export const itemTile = createTile({
  type: ['hn_api', 'item'],
  fn: ({ api, params }) => api.child(`item/${params.id}`).once('value').then(snapshot => snapshot.val()) ,
  nesting: ({ id }) => [id],
  caching: true,
});
```

正如您所看到的，这个实现非常类似于前一个项目`storiesTile`。事情是他们正在执行几乎相同的操作 – 请求一些API调用，解析结果（如果需要的话 - 这里我们不必），设置嵌套和缓存 - 事实上，这是应该的。`redux-tiles`背后的思想是，由于没有样板代码，创建代表原子功能的小“tile”非常便宜，然后将其组合起来。
让我们为下载项的列表创建tile——没有这样的端点，所以我们必须组合现有的tile:
```javascript
export const itemsTile = createTile({
  type: ['hn_api', 'items'],
  fn: ({ dispatch, actions, params }) =>
    Promise.all(params.ids.map(id =>
      dispatch(actions.hn_api.item({ id }))
    ))
});
```

我们只是迭代id并请求里面的所有ID。如果我们只想执行一定数量的同时请求，我们可以在这里对这些请求进行分块（并且对于其他的tile将被完全抽象）。

最后，现在我们可以创建按类型返回stories的分页功能。 逻辑如下：
- 获取这个类型的所有stories的列表
- 计算给定页面的id
- 下载这些ID的所有项目
- 将它们嵌套到`[type，pageNumber，pageSize]`中

```javascript
export const itemsByPageTile = createTile({
  type: ['hn_api', 'pages'],
  fn: async ({ params: { type = 'topstories', pageNumber = 0, pageSize = 30 }, selectors, getState, actions, dispatch }) => {
    // 我们总是可以抓取stories，缓存它们，所以如果这个类型已经被抓取了，就不会有新的请求
    const { data } = await dispatch(actions.hn_api.stories({ type }));
    const offset = pageNumber * pageSize;
    const end = offset + pageSize;
    const ids = data.slice(offset, end);
    
    // 下载给定页面的ID列表
    await dispatch(actions.hn_api.items({ ids }));
    
    // 用实际的值填充id
    return ids.map(id => selectors.hn_api.item(getState(), { id }).data);
  },
  // 我们可以以这种方式安全地嵌套它们，并确保单个项目将被缓存
  // 因此，更改页面上的条目数字可能甚至不需要一个新的请求
  nesting: ({ type = 'topstories', pageNumber = 0, pageSize = 50 }) => [type, pageNumber, pageSize],
});
```

最后一个tile包含我们应用程序的主要业务逻辑，但它不包含任何直接的API请求，所以如果将来某个端点的响应会改变，或者我们将不得不做不同的请求来获得相同的数据，那么我们只能在这些小tile（解析数据或者调度其他tile）内部改变它。

现在让我们把它放在一起 - 我们将需要创建所有的实体，然后用reducer和middleware来创建redux store。

```javascript
import { createStore, applyMiddleware } from 'redux';
import { createEntities, createMiddleware } from 'redux-tiles';

const tiles = [
  storiesTile,
  itemTile,
  itemsTile,
  itemsByPageTile
];

// 我们只从redux-tiles中创建store，所以我们不必指定第二个参数，这是store中的一个名称空间
const { actions, reducer, selectors } = createEntities(tiles);

// 稍后我们需要`waitTiles`来等待所有的请求
const { middleware, waitTiles } = createMiddleware({ api, actions, selectors });

const store = createStore(
  reducer,
  applyMiddleware(middleware)
);
```

现在我们可以下载我们的头版新闻！

```javascript
// 下载topstories的第一页
store.dispatch(actions.hn_api.pages({ type: 'topstories' }));

// 等待所有的请求 - 在这里只是一个单一的
await app.waitTiles();

// 让我们来check我们下载了30个
const { data } = app.selectors.hn_api.pages(store.getState(), { type: 'topstories' });
assert(data.length, 30); // will be true!
```

## 实现链接

- [complete code](https://github.com/Bloomca/redux-tiles/tree/master/examples/hacker-news-api)
- [tiles code](https://github.com/Bloomca/redux-tiles/blob/master/examples/hacker-news-api/hn-tiles.js)
- [tests code](https://github.com/Bloomca/redux-tiles/blob/master/examples/hacker-news-api/__test__/app.spec.js)