# 与现有的Redux store集成

尽管在大多数情况下，redux-tiles应该足够用于整个redux store，但它有一些缺点（特别是在复杂的异步交互中），也许您想将其添加到现有项目中，或者必须与另一个库他们自己的reducer合并。这并不难，你只需要做更多的行为来实现它。

## Reducer

你必须做的第一件事 － [结合reducer](http://redux.js.org/docs/recipes/reducers/UsingCombineReducers.html)。 您从[createEntities](../api/createEntities.md)或[createReducers](../api/createReducers.md)创建的Reducer将不在顶层 - 它将会降低一级，所以我们必须指定它的名称，以便所有选择器都能正常工作：

```js
import { createEntities, createReducers } from 'redux-tiles';
import tiles from '../tiles';

const { reducer: reduxTilesReducer } = createEntities(tiles, 'redux_tiles');
// or
const reduxTilesReducer = createReducers(tiles);
```

现在所有的选择器都会知道，为了找到数据，他们必须更深入一层。 让我们把这个reducer与其他的reducer结合：

```js
import { combineReducers, createStore } from 'redux';

const finalReducer = combineReducers({
  ...otherReducers,
  redux_tiles: reduxTilesReducer,
});
```

就这样，我们得到了可以传递给`createStore`函数的reducer，以及初始state和中间件：

```js
import { createStore, applyMiddleware } from 'redux';
import { createMiddleware } from 'redux-tiles';
import tiles from '../tiles';

const { middleware } = createMiddleware(tiles);
const store = createStore(
  finalReducer,
  // 恢复客户端上的数据，通常是类似window.__ INITIAL_STATE__的东西
  initialState,
  applyMiddleware(middleware)
);
```

## Actions and Selectors

对于action和selector，我们并不需要真的做任何事情 – 在我们将正确的名称空间传递给`createReducers`或`createEntities`后，它将使选择器正常工作。
行动不受影响，我们只需要将它们整合到更大的action和selector对象中：

```js
import { createEntities } from 'redux-tiles';
import tiles from '../tiles';

const {
  actions: tilesActions,
  selectors: tilesSelectors,
  reducer: tilesReducer
} = createEntities(tiles, 'redux_tiles');

export const actions = {
  ...otherActions,
  tiles: tilesActions,
};

export const selectors = {
  ...otherSelectors,
  tiles: tilesSelectors,
};
```
