# CreateSyncTile API

CreateSyncTile与[createTile](./createTile.md)非常相似，但不提供异步功能。因此，它有点简单 - 没有缓存，只有两个常量 - SET和RESET。

基本的同步tile将如下所示：
```javascript
import { createSyncTile } from 'redux-tiles';

const nofitications = createSyncTile({
  type: ['ui', 'notifications'],
  实际上，如果您只是将params传入store，那么您可以省略它——默认函数正是这个功能
  fn: ({ params }) => params,
});
```

因为在引擎盖下没有async操作，也没有相关的数据结构 – 从函数返回的内容将直接进入状态（并从选择器的函数中返回）。

Let's take a look at a little bit more extensive example:

```javascript
import { createSyncTile } from 'redux-tiles';

const chooseParams = createSyncTile({
  // type的工作原理与`createTile`完全相同–将嵌套在store中，
  // 并且`createEntities`将把它放在actions/selectors中的这个命名空间下
  type: ['ui', 'params'],

  // 而你只需要执行同步操作来返回一些值，你可以dispatch其他的action，但要小心，
  // 这个函数将在actions.ui.params下可用
  // 因此，例如，您可以执行自动关闭通知（例如，调度相同的tile action），或发送一些跟踪 – 
  // 它很容易发起一些race-conditions – 对于复杂的异步的东西，可以考虑使用像RxJS或redux-saga这样的东西
  fn: ({ params }) => {
    return params.data;
  },
  // 或者，如果您对现有数据执行某些操作，
  // 编写更多的声明性操作可能是有用的
  // 它们具有完全相同的签名，并将返回的数据分发给tile;
  // 它们将在actions.ui.params.add和actions.ui.params.remove下提供
  fns: {
    add: ({ params, selectors, getState}) => {
      const list = selectors.notifications(getState(), params);
      return list.concat(params);
    },
    remove: ({ params, selectors, getState}) => {
      const list = selectors.notifications(getState(), params);
      return list.filter(({ id }) => id !== params.id);
    },
  },

  // 和`createTile`一样 - 独立的数据，所以它们将独立存储在store里面
  nesting: ({ type }) => [type],

  // 初始状态放入reducer。 如果存储元素列表，
  // 并且不想检查数据类型是数组还是对象，
  // 使用此属性
  initialState: [],
});
```

## 范例

我们来为todoMVC应用程序创建一个tile。我们必须执行很多被触发通知的列表清单。为了简单起见，我们不会在这里删除它们，但是我们可以通过过滤来轻松地添加它们。

```javascript
import { createSyncTile } from 'redux-tiles';

export const todosTile = createSyncTile({
  type: ['todos', 'list'],
  // 为了创建更多的声明式函数名称，我们将使用fns属性。
  // 来自这个对象的所有函数都会被附加到主action上
  // 所以他们将在actions.todos.list.%function%下被提供
  fns: {
    // 这个函数将在actions.todos.list.add下可用
    add: ({ params, getData }) => {
      // getData是一个返回当前值的特殊函数
      // 它相当于以下内容：
      // const list = selectors.todos.list(getState());
      const list = getData();
      const newItem = {
        ...params,
        completed: false,
        id: Math.random(),
      };

      return list.concat(newItem);
    },
    // 这个函数将在actions.todos.list.remove下可用
    remove: ({ params, selectors, getState }) => {
      const list = selectors.todos.list(getState());
      return list.filter(item => item.id !== params.id);
    },
    // 这个函数将在actions.todos.list.toggle下可用
    toggle: ({ params, selectors, getState }) => {
      const list = selectors.todos.list(getState());
      return list.map(todo => todo.id === params.id
        ? { ...todo, completed: !todo.completed }
        : todo
      );
    }
  },
  // 我们将初始状态表示为[]，所以我们不必检查数据的类型（我们确信它将是一个数组）
  initialState: [],
});
```

## 更多信息

* [Nesting](../advanced/nesting.md)
* [Selectors](../advanced/selectors.md)