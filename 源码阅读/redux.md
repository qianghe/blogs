## Redux源码解析

### 前提
如果你还是不熟Redux的使用姿势，建议先看下[官方文档](https://redux.js.org/introduction/getting-started)。

### 多言几句
在使用一样工具之前，总是会想到”它“出现的缘由是什么？解决的问题又是什么？工具的优劣势分别是什么，是都能帮助现有项目解决问题？是都会带来衍生问题？...

俗话说：”方法总比困难多“，那就简单速览下Redux吧。

说起Redux，不得不提起Facebook团队的另一个框架[Flux](https://facebook.github.io/flux/)。同MVC模型不同的是，Flux提出的数据流处理模型中是没有Controller的，可以把它看作是controlled-view。Flux的工作模型如下：


![Flux workflow](https://github.com/qianghe/blogs/blob/main/imgs/flux-workflow.png?raw=true)

那既然有了Flux了，为什么还需要Redux？因为Flux其相对的“复杂性”：

* 调试定位困难
   * Flux支持多store，store之间可能存在关联关系，触发依赖action；
   * Flux中的store中注册的dispatch需要支持异步处理（提供了waitFor API），可能存在不可预测的异步结果；
***

基于以上的原因，有了简化版的Redux（我掐指猜测了一下，这个名字是Reduced Flux的意思吧...）：

* Redux只有一个全局store;
* Redux通过reducer处理所有的变更，并且都是纯函数，避免了边际效应；

因此可以说Redux提供的dispatch(action)都是可预测的行为，保障了我们应用的行为边界预测性，便于测试和debug。

但任何工具的引入都会存在利弊，这就涉及到在什么场景下我们需要Redux。当然，官方文档都写的很明白了，[查看](https://redux.js.org/tutorials/essentials/part-1-overview-concepts#when-should-i-use-redux)。

![When should use redux](https://github.com/qianghe/blogs/blob/main/imgs/when-shoul-use-redux.png?raw=true)


### 源码

redux的源码很少，因为其功能单一且独立（纯js，独立于框架）。
可以想一下如果我们要实现一个这样库，需要做哪些工作呢？

* 创建一个store实例 -> createStore()：这个store提供了一些功能，比如getState()/dispatch/subscribe。
* 支持middlewares（中间件）扩展能力：中间件让redux有了更为强大的能力，比如redux-thunk使得dispatch支持了异步。

##### combineReducer

当我们在写一个reducer的时候，它是一个纯函数，逻辑是这样的：

`reducer(oldState, action) => newState`

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

`dispatch(action) => action`

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



