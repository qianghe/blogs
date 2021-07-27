## Redux源码解析

### 前提

如果你还是不熟Redux的使用姿势，建议先看下[官方文档](https://redux.js.org/introduction/getting-started)。

### 多言几句

在使用一样工具之前，总是会想到”它“出现的缘由是什么？解决的问题又是什么？工具的优劣势分别是什么，是都能帮助现有项目解决问题？是都会带来衍生问题？...

俗话说：”方法总比困难多“，那就简单速览下Redux吧。

说起Redux，不得不提起Facebook团队的另一个框架[Flux](https://facebook.github.io/flux/)。同MVC模型不同的是，Flux提出的数据流处理模型中是没有Controller的，可以把它看作是controlled-view。Flux的工作模型如下：

<img src="https://github.com/qianghe/blogs/blob/main/imgs/flux-workflow.png?raw=true" alt="Flux workflow" style="zoom:60%;" />

那既然有了Flux了，为什么还需要Redux？因为Flux其相对的“复杂性”：

* 调试定位困难
  * Flux支持多store，store之间可能存在关联关系，触发依赖action；
  * Flux中的store中注册的dispatch需要支持异步处理（提供了waitFor API），可能存在不可预测的异步结果； 

基于以上的原因，有了简化版的Redux（我掐指猜测了一下，这个名字是Reduced Flux的意思吧...）：

* Redux只有一个全局store;
* Redux通过reducer处理所有的变更，并且都是纯函数，避免了边际效应；

因此可以说Redux提供的dispatch(action)都是可预测的行为，保障了我们应用的行为边界预测性，便于测试和debug。

但任何工具的引入都会存在利弊，这就涉及到在什么场景下我们需要Redux。当然，官方文档都写的很明白了，[查看](https://redux.js.org/tutorials/essentials/part-1-overview-concepts#when-should-i-use-redux)。

![When should use redux](https://github.com/qianghe/blogs/blob/main/imgs/when-shoul-use-redux.png?raw=true)

### 源码

redux的源码很少，因为其功能单一且独立（纯js，独立于框架）。

可以想一下如果我们要实现一个这样库，需要做哪些工作呢？

1. 创建一个store实例 -> createStore()：这个store提供了一些功能，比如getState()/dispatch/subscribe。
2. 支持middlewares（中间件）扩展能力：中间件让redux有了更为强大的能力，比如redux-thunk使得dispatch支持了异步。

##### combineReducer

当我们在写一个reducer的时候，它是一个纯函数，逻辑是这样的：

```
reducer(oldState, action) => newState
```

当我们有多个reducer的时候，就需要将它聚合起来，通过combineReducer处理，其实返回了一个函数，这个函数的形态（输入、输出）同一个单独的reducer很像，其在内部调用了所有的reducer用以计算新的newState：

```javascript
export default function combineReducers(reducers) {
  const finalReducers = {}
  //...统计了有效的reducer
  const finalReducerKeys = Object.keys(finalReducers)

  //...
  return function combination(state, action) {
    // ...
    let hasChanged = false
    const nextState = {}
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      const previousStateForKey = state[key]
      const nextStateForKey = reducer(previousStateForKey, action)
      if (typeof nextStateForKey === 'undefined') {
        // ... 错误提醒
      }
      // 更具当前的reducer计算key对应的state部分
      nextState[key] = nextStateForKey
      // 只要又一次的reducer计算导致state的变化，就认为是state更新了
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    hasChanged =
      hasChanged || finalReducerKeys.length !== Object.keys(state).length
    return hasChanged ? nextState : state
  }
}
```

##### 基础的dispatch

redux本身提供的dispatch函数起形态如下：

```
dispatch(action) => action
```

其中action只能为object对象: { type: xxx, ... }，可见其功能也很单一，通过用户触发的action，以计算新的state：

```javascript
function dispatch(action) {
  //...异常处理
  try {
    isDispatching = true
    // 根据action计算“新”的state
    currentState = currentReducer(currentState, action)
  } finally {
    isDispatching = false
  }
  // 如果有subscribe的订阅者，则执行它们的回调
  // nextListeners是在subscribe中动态收集的数组
  // currentLiseners是在执行dispatch事被调用的数组，在此时进行赋值
  const listeners = (currentListeners = nextListeners)
  for (let i = 0; i < listeners.length; i++) {
    const listener = listeners[i]
    listener()
  }

  return action
}
```

##### 进阶的dipsptch

redux提供的dispatch只支持object对象的方式同步修改state的值，那如果需要异步处理的流程，即dispatch一个函数是否是可行的呢？那这时候就需要我们通过某个方式去改写dipsatch，用以扩展支持function(异步)的情况。还记得前面我们说到，redux本身是纯函数的集合，功能单一，需要扩展功能的话，就需要使用middlewares，即中间件：通过获取用户action来做自定义的行为，比如我们说的redux-thunk、redux-logger等。

官方代码对createStore中第三个参数enhancer的解释：

```javascript
 * @param enhancer The store enhancer. You may optionally specify it
 * to enhance the store with third-party capabilities such as middleware,
 * time travel, persistence, etc. The only store enhancer that ships with Redux
 * is `applyMiddleware()`.
```

那多个middleware串行工作，就需要一种方式，将任务链关联到一起。middleware的格式化模版如下：

```
({ dispatch, getState }) => next => action => { ... return next(action) }
```

而将任务关联串行的主要工具函数就是compose:

```javascript
compose(...funcs: Function[]) {
  // ...校验
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

通过compose函数组合我们可以获知：上一个插件的next的执行函数其实是下一个插件的 (action) => {}，通过这种方式将任务串联化。


applyMiddleware的作用就是通过compose函数构造一条新的dispatch链：

```javascript
applyMiddleware(...middlewares) {
  return (createStore) => (reducer, preloadedState) => {
      // 在appliMiddleware中构造store才能获取到原始的dispatch
      const store = createStore(reducer, preloadedState)
      let dispatch = () => {
        //...error deal
      }
      const middlewareAPI = {
        getState: store.getState,
        dispatch: (action, ...args) => dispatch(action, ...args)  
      }
      // ({ getStage, dispatch })注入
      const chain = middlewares.map(middleware => middleware(middlewareAPI))
      // 串联到一起
      dispatch = compose(...chain)(store.dispatch)
      return {
        ...store,
        dispatch
      }
    }
}
```
（理解这一块的更好的方式，是自己写几个middleware，体验一下就能更快的理解了）。

### 总结
1. redux出现的意义及其使用场景；
2. redux是函数式编程的体现，其是纯函数；
3. redux的几个关键：
   需要管理什么？数据state(通过reducer计算合并) ->
   需要更新什么？通过dispatch(action)更新数据从而更新视图 ->
	 如何扩展dispatch为异步？通过redux-thunk等中间件起到加强的作用 ->
	 如果扩展redux功能？通过middleware插件实现对logger等。