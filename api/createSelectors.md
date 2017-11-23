# createSelectors API

正如在[createTile](./createTile.md)中所陈述的，选择器对象是暴露的，可以毫无问题地查询状态。但是由于与 [createActions](./createActions.md)中所描述的相同的原因，拥有与`type`属性相同的嵌套是很好的，这个函数会关心它：

```javascript
import { createSelectors } from 'redux-tiles';
import tiles from '../tiles';

const selectors = createSelectors(tiles);
```

## 注意最上面的reducer

如果将redux-tiles集成到现有项目中，或者使用其他non-tiled reducer，则必须在[createEntities](./createEntities.md)或[createReducers](./createReducers.md)中指定命名空间，以便选择器将尝试获取正确的数据。

## Tiles 参数

`createActions`只有一个参数，`tiles`，它可以是一个数组或者一个包含tile的对象。 下一个结构是允许的：

```javascript
const userTiles = [userLogin, userData, userPreferences];
const uiTiles = [notifications, popup];
const arrayTiles = [
  ...userTiles,
  ...uiTiles,
];

const objectTiles = {
  userLogin,
  userData,
  userPreferences,
  notifications,
  popup,
};
```

要了解有关实际选择器功能的更多信息，请阅读[高级指南](../advanced/selectors.md)。 它解决了为什么我们需要这样的功能以及如何使用它们这样的问题。
