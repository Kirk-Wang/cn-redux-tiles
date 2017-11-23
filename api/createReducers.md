# createReducers API

在这个库中，reducer非常复杂 - 为了正确选择路径，我们必须指定在哪个命名空间下的reducer将被组合。默认情况下它是空的，只有把你的tile reducer放在顶层时，这种情况才适用。但是，如果要集成到现有项目或其他reducer，则必须将此名称空间指定为第二个参数。

```javascript
import { createReducers } from 'redux-tiles';
import tiles from '../tiles';

// 如果您使用_only_ redux-tiles，请随时保留它
// 在顶层，只是省略第二个参数
const reducer = createReducer(tiles, 'redux_tiles');
```

有人可能会问，为什么我们在这里指定命名空间，如果它只影响选择器?我们将reducer结合在一起，我们必须提供正确的命名空间，因此在这里提供更正确——这是一个原因，而不仅仅是结果，就像选择器一样。

## Tiles 参数

`createActions`只有一个参数，`tiles`，它可以是一个数组或者一个包含tile的对象。 下一个结构是允许的：:

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
