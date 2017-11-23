# CreateActions API

正如在[createTile](./createTile.md)中所描述的那样，您可以直接从tile中获取action，并自行dispatch，根本没有任何警告。但是，一个问题在于嵌套的地方 – 我们可以指定`type: ['ui', 'notifications', 'warning']`，但是如果我们直接访问action，就必须自己结合一些共同的对象来实现类似`actions.ui.notifications.warning`的事情。`createActions`正好解决了这个问题 - 它遍历了tile数组，并基于给定的type构造了这个嵌套对象。

So, let's see some example:

```javascript
import { createActions, createTile } from 'redux-tiles';

const userTile = createTile({
  type: ['user', 'data', 'get'],
  fn: someAction,
});
const tiles = [userTile];

const actions = createActions(tiles);
actions.user.data.get === userTile.action; // true
```
这个函数是一个纯sugar，但是它们的嵌套方式与`type`属性完全一致，这是非常好的。

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