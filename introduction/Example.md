# 范例

这是一个如何与库一起工作的例子。 这是一个非常简单的例子。 如果您需要更具体的内容，请参阅[示例](../examples/README.md)或[API部分](../api/README.md)的某些特定部分。 我们将构建一个简单的store来获取文档，在我们获取它们之后，我们将显示通知5秒钟，然后关闭它。

```javascript
import { createEntities, createTile, createSyncTile, createMiddleware } from 'redux-tiles';
import { createStore, applyMiddleware } from 'redux';
import { sleep } from 'delounce';
import api from '../api';

// async tile，它将查询API，并将自动填充
// isPending，正在处理的时候，并把结果放到`data`或`error`中
// field，取决于promise如何resolved
const documents = createTile({
  type: ['documents', 'api'],
  fn: ({ api, params }) => api.fetch(`/documents/${params.type}`),
  // 我们按类型嵌套文档，所以请求不同的类型是独立的
  nesting: ({ type }) => [type],
});

// sync tile，它不执行任何异步操作。 事实上，我们
// 仍然可以分发异步action，但是我们不允许等待
// 他们 - 这个函数的结果将被置于相应的状态
const notifications = createSyncTile({
  type: ['ui', 'notifications'],
  fn: ({ params }) => params.data,
  nesting: ({ type }) => [type],
});

// 此tile将是async，但它不会执行直接api请求
// 相反，它会分派其他的tile;redux-tiles鼓励你
// 把你的逻辑分成小部分，所以很容易把它结合起来
// 并且稍后更换/删除零件
const fetchDocumentWithNotification = createTile({
  type: ['documents', 'fetchWithNotification'],
  fn: async ({ dispatch, actions, params: { type } = {} }) => {
    // 我们分发其他tile，而不是在这里下载文件 – 
    // 它使我们能够更容易地组合我们逻辑的小部分
    await dispatch(actions.api.documents({ type }));
    const data = `Document ${type} was downloaded!`;
    dispatch(actions.ui.notifications({ type, data }));
    
    await sleep(params.timeout || 5000);
    // 发送没有数据的type，我们将关闭通知
    dispatch(actions.ui.notifications({ type: params.type }));

    return { ourData: 'some' };
  },
  nesting: ({ type }) => [type],
});

const tiles = [documents, notifications, fetchDocumentWithNotification];

const { acttons, reducer, selectors } = createEntities(tiles);
const { middleware, waitTiles } = createMiddleware({ actions, selectors });
const store = createStore(reducer, applyMiddleware(middleware));

// 我们只分发最复杂的action - 这个tile就是这个应用程序的“本质”，
// 它只组成其他tile，它们有自己的单一责任 - 获取文档和显示通知
store.dispatch(actions.documents.fetchWithNotification({ type: 'terms' }));

// 这个函数将等待所有分发的action，这可能会有助于服务端渲染，等待所有的请求
await waitTiles();
// 让我们发送另一个通知，来看看
store.dispatch(actions.ui.notifications({ type: 'agreement', data: 'Our agreement!' }));

// 让我们打印我们的最终store
console.log(JSON.stringify(store.getState(), null, 2));
/*

{
  documents: {
    api: {
      terms: {
        isPending: false,
        fetched: true,
        data: {
          url: 'https://example.com/terms.pdf',
          size: 512
        },
        error: null,
      },
    },
    fetchWithNotification: {
      terms: {
        isPending: false,
        fetched: true,
        data: { ourData: 'some' },
        error: null,
      },
    },
  },
  ui: {
    notifications: {
      terms: undefined,
      agreement: 'Our agreement!',
    },
  },
}
*/
```
