# Redux样板代码

虽然这个库的目标是消除这个样板，但了解它如何在引擎盖下工作是至关重要的 - 没有魔法，它只是一个方便的抽象;但是如果不理解，可能会遇到一些麻烦，并会尝试用奇怪的方式解决一些问题。所以，不要忽视基本的东西 - 也有很多关于这个话题的好资源，正是来自于Redux的创建者（例如免费视频课程，[1](https://egghead.io/series/getting-started-with-redux)和[2](https://egghead.io/courses/building-react-applications-with-idiomatic-redux)）。[官方文档](http://redux.js.org/)也很棒--在进入这个库之前先看看吧!

所以，为了快速重清Redux的原理，让我们简单地创建一个小模块。但首先让我们回顾一下总体流程：redux具有单向的数据流，在开始时看起来有点怪异 – 我们创建常量作为唯一的标识符，然后通过与这个常量一起的action传递数据，然后我们把这个数据放在store中它自己的位置（每个处理程序只对特定的常量作出反应）。此外，只能从store实例触发这些action，所以它是由我们的store（在React的情况下存储在上下文中）“批准”的，因此我们肯定知道这些数据来自哪里。以后,组件（或只是应用程序的某些部分 – 你可以手动订阅store的某些部分）只是检索这些数据，他们没有权力改变状态。

所以，考虑到这些知识，让我们在Redux中构建一个简单的通知模块。 假设我们有预定义的通知列表，我们只是将数据传递给它，或者将其设置为null，以防万一关闭它（还记得这些用来接受cookie的恼人的弹出窗口吗?）。

我们将从action开始：
```javascript
export const ADD_NOTIFICATION = 'ADD_NOTIFICATION';
export const REMOVE_NOTIFICATION = 'REMOVE_NOTIFICATION';

export const addNotification = ({ notification, id }) => {
  return {
    type: ADD_NOTIFICATION,
    payload: { notification, id },
  };
};

export const removeNotification = (id) => {
  return {
    type: REMOVE_NOTIFICATION,
    payload: { id },
  },
};
```

然后是 reducer:
```javascript
import { ADD_NOTIFICATION, REMOVE_NOTIFICATION } from './actions';

const initialState = {
  cookies: null,
  login: null,
  details: null,
};

export const reducer = (state = initialState, action) {
  switch (action.type) {
    case ADD_NOTIFICATION: {
      const { id, notification } = action.payload;
      return {
        ...state,
        [id]: notification,
      }
    };
    case REMOVE_NOTIFICATION: {
      const { id } = action.payload;
      return {
        ...state,
        [id]: null
      }
    };
    default:
      return state;
  }
}
```

那么，这里有什么问题？ 看起来并不是很长，并且我们现在已经保证在我们的用户界面一致的更新！那么问题是，对于真正的小应用程序来说是好的，但通常随着应用程序的增长，它会变得到处都是 – 因为它是很多样板代码，最终会有一个很大的可能性，那就是旧的　action/reducer　将会被扩展，而不是创创建小的可组合的piece。此外，combineReducers通常只在顶层执行，因此您最终将在单个reducer中结合多个UI状态，或者类似于`notificationsUI`这样的其他类型。

考虑前面的例子，但是现在用了redux-tiles：
```javascript
import { createSyncTile } from 'redux-tiles';

const notifications = createSyncTile({
  type: ['ui', 'notifications'],
  fn: ({ params: { notification = null } }) => notification,
  nesting: ({ id }) => [id],
});
```
你会得到相同的常量，很好的嵌套功能，你的reducer状态将被很好地放在`state.ui.notifications`下。

最后，我可以推荐一些关于Redux冗长的牢骚，并分析为什么如此：
- [SOME PROBLEMS WITH REACT/REDUX by André Staltz](https://staltz.com/some-problems-with-react-redux.html)
- [Idiomatic Redux by Mark Erikson](http://blog.isquaredsoftware.com/2017/05/idiomatic-redux-tao-of-redux-part-1/) – pretty good resource to deep your knowledge about redux itself
- [Practice and Philosophy of Redux by Mark Erikson](http://blog.isquaredsoftware.com/2017/05/idiomatic-redux-tao-of-redux-part-2/) – analyzing of each part of Redux