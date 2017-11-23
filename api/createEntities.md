# createEntities API

所有这三个实体（[createActions](./createActions.md)，[createReducers](./createReducers.md)和[createSelectors](./createSelectors.md)）的函数都有非常相似的API--所以，我们可以不写五六个代码，而是写两个。这就是这个函数的全部概念，下面的例子说明了它：

```javascript
import { createEntities } from 'redux-tiles';
import tiles from '../tiles';

// 如果你使用_only_redx-tiles，可以随意保留在顶层，只是省略第二个参数
const { actions, reducer, selectors } = createEntities(tiles, 'redux_tiles');
```

## 第二个参数

第二个参数（`redux_tiles`在这里）是可选的，但是如果你想把你的reducer放在某个名字空间下（所以把它和其他的reducer集成在一起）是必须的。在这种情况下，它将执行这个名字空间下的所有选择器，为了正确的工作，你必须把这个reducer结合在这个键下：

```javascript
import { combineReducers } from 'redux';
import { reduxTileReducer } from '../entities';
import reducers from './reducers';

const reducer = combineReducers({
  ...reducer,
  redux_tiles: reduxTileReducer,
});
```

## Tiles 参数

`createActions`只有一个参数，`tiles`，它可以是一个数组或者一个包含tile的对象。 下一个结构是允许的：

```javascript
const userTiles = [userLogin, userData, userPreferences];
const uiTiles = [notifications, popup];
const arrayTiles = [
  ...userTiles,
  ...uiTiles
];

const objectTiles = {
  userLogin,
  userData,
  userPreferences,
  notifications,
  popup
};
```
