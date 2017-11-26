# 选择器

我们将所有的数据都保存在redux state - 所有tile都将它们的数据保存在`type`命名空间下。 这意味着你可以很容易地直接从state对象访问你的数据：

```js
import { createTile, createEntities } from 'redux-tiles';

const exampleTile = createTile({
  type: ['example', 'nested', 'tile'],
  ...
});

const tiles = [exampleTile];

// 我们为tile指定命名空间，这意味着我们在这个key下将reducer与tileReducer结合起来
const { actions, reducer, selector } = createEntities(tiles, 'redux_tiles');

// 我们在actions对象中获得这个嵌套
dispatch(actions.example.nested.tile());

// 要访问数据，我们可以使用与顶级名称相同的嵌套
// 另外，请注意，这将返回null（如果我们还没有从这个tile派发action），
// 为了与嵌套tile一致 - 我们不能预填充数据的嵌套值
const result = state.redux_tiles.example.nested.tile;
```

所以模式很清楚，对吧？ 所以我们可以直接访问我们所有的数据！
那么，别太快。 让我们考虑嵌套，然后再次尝试访问我们的示例tile：


```js
const exampleTile = createTile({
  type: ['example', 'nested', 'tile'],
  nesting: ({ type, query }) => [type, query],
  ...
});

// 让我们尝试访问我们的数据，通过type和query
// 这个查询可能实际上会引发错误！
// 默认情况下，state只包含一个空对象，
// 并且所有嵌套的值都将被呈现_only_，以防我们用这样的参数dispatch一些action！
const result = state.redux_tiles.example.nested.tile.myType.myQuery;
```

所以，嵌套变得有点棘手 - 我们不得不使用像[lodash中的get方法](https://lodash.com/docs/4.17.4#get)，它可以安全地通过属性访问;而且，使用这种访问方式，我们将不得不经常检查值是否已初始化，并在所有地方添加默认值。
另外，如果我们想改变reducer的命名空间，我们将不得不花费一些时间来查找所有的事件。

为了解决所有这些问题并抽象数据访问，我们引入了_selectors_的概念。Selector只是一个函数，它以某种方式检索您的数据 - 我们强烈建议您编写自己的selector！如果你觉的在一些事情像“从application获取id，如果没有，转到最新的记录并且如果再次没有好运，采取分配和查询他们”上重复着。将所有这些功能移到`getId`选择器是一个好迹象。推荐使用selectors的方法是通过`createSelectors`或`createEntities`来组合它们：

```js
import { createSelectors, createEntities } from 'redux-tiles';
import tiles from '../tiles';

const selectorsFromCreateSelectors = createSelectors(tiles);
const { selectors, actions, reducer } = createEntities(tiles);

expect(selectorsFromCreateSelectors).deepEqual(selectors); // true
```

selectors具有与actions相同的嵌套结构，因此可以通过`selectors.some.type`访问类型为`['some', 'type']`的tile。Selector函数有两个参数 – 第一个是state，第二个是parameters（和你传递作为第一个参数去分发tile的action一样）- 这个东西实际上会给你当前的状态或默认值，这是下面的数据结构:

```js
{
  isPending: false,
  fetched: false,
  data: null,
  error: null,
}
```

没有同步tile的默认值，因为我们在那里没有任何元数据，我们能够保留任何东西（除了`undefined`，因为它不被redux store 接受）。所以它的选择器只解决了访问嵌套属性和reducer命名空间的问题。如果你想传递默认值到同步tile，使用`initialState`属性。

另外，你可以在异步tile中保留任何`data`，甚至是`null`或`undefined` - `fetched`字段允许你检查请求是否继续。

使用[connect](https://github.com/reactjs/react-redux/blob/master/docs/api.md#connectmapstatetoprops-mapdispatchtoprops-mergeprops-options) 的典型代码如下所示：

```js
// 直接访问：
const mapStateToProps = (state, { params: { id } } => ({
  // 在缺失值的情况下将是undefined
  item: state.tiles.api.item[id],
});

// and using selectors
const mapStateToProps = state => ({
  // 在缺失值的情况下将是
  // {
  //   isPending: false,
  //   fetched: false,
  //   error: null,
  //   data: null,
  // }
  item: selectors.api.item(state, { id }),
});

export default connect(mapStateToProps)(Component);
```

## Why state without nesting by default is null?

As it was mentioned previously, by default tile without nesting sets it's value to null. While it is possible to set it to the typical data structure (for async tile, where we have metadata):

```js
{
  isPending: false,
  fetched: false,
  data: null,
  error: null,
}
```

We don't do it absolutely _intentionally_. The problem is that we'd get inconsistency with data, which is taken from tiles with nesting (we cannot guess which values will be there! Of course, we could add intrinsic getter, but it will make things much more complicated.) – so if you access data by hand, then at least you'd get _almost_ consistent result (almost – with nesting, you'll have have to specify your fallback in case of no value).

Once again, the recommended way to go is to use selectors, provided by tiles (and combined by `createSelectors` or `createEntities`).