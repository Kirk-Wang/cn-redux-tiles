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
    // we can always fetch stories, they are cached, so if this type
    // was already fetched, there will be no new request
    const { data } = await dispatch(actions.hn_api.stories({ type }));
    const offset = pageNumber * pageSize;
    const end = offset + pageSize;
    const ids = data.slice(offset, end);
    
    // download list of ids for given page
    await dispatch(actions.hn_api.items({ ids }));
    
    // populate ids with real values
    return ids.map(id => selectors.hn_api.item(getState(), { id }).data);
  },
  // we can safely nest them this way, and be sure that individual items will be cached
  // so, changing number of items on the page might not even require a single new request
  nesting: ({ type = 'topstories', pageNumber = 0, pageSize = 50 }) => [type, pageSize, pageNumber],
});
```

The last tile contains main business logic for our application, but it does not contain any direct api request, so if in the future response for some endpoint will change, or we will have to do different requests to get the same data, we can change it only inside these small tiles (parsing data or dispatching other small tiles).

Let's put it together now – we will need to create all entities, and then redux store with reducer and middleware.

```javascript
import { createStore, applyMiddleware } from 'redux';
import { createEntities, createMiddleware } from 'redux-tiles';

const tiles = [
  storiesTile,
  itemTile,
  itemsTile,
  itemsByPageTile
];

// we create store only from redux-tiles, so we don't have to specify
// second argument, which is a namespace in the store
const { actions, reducer, selectors } = createEntities(tiles);

// we will need `waitTiles` later to wait for all requests
const { middleware, waitTiles } = createMiddleware({ api, actions, selectors });

const store = createStore(
  reducer,
  applyMiddleware(middleware)
);
```

And now we can download our front page with top stories!

```javascript
// download first page of topstories
store.dispatch(actions.hn_api.pages({ type: 'topstories' }));

// wait all requests – here it is just a single one
await app.waitTiles();

// let's check that we downloaded 30 stories
const { data } = app.selectors.hn_api.pages(store.getState(), { type: 'topstories' });
assert(data.length, 30); // will be true!
```

## Links to implementation

- [complete code](https://github.com/Bloomca/redux-tiles/tree/master/examples/hacker-news-api)
- [tiles code](https://github.com/Bloomca/redux-tiles/blob/master/examples/hacker-news-api/hn-tiles.js)
- [tests code](https://github.com/Bloomca/redux-tiles/blob/master/examples/hacker-news-api/__test__/app.spec.js)