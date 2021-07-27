## Vuex 源码学习

##### 文档

https://vuex.vuejs.org/zh/

Vuex灵感来自于Flux，它们都是“单根”（而为什么是单根的，我想可能是为了明确一个mutation或者action来自于一个明确的module吧）

为了维护庞大的store，可以通过modules拆分进行状态维护。

看下Module的定义：

![截屏2021-06-04 上午11.52.01.png](https://cdn.nlark.com/yuque/0/2021/png/388960/1622778746132-1086ceff-f36c-4410-a55e-2ed38179098d.png?x-oss-process=image%2Fresize%2Cw_1000)

从其属性_children我们看出，module其实可以被“无限”嵌套。

如果module定义了namespaced： true（为了使模块更为功能独立），那么在调用时需要指定module的path；

如果namespaced: false，那么如果多个module定了同名mutation， commit时会同时触发（但子module有其独立的state）；

比如我们定义一个store如下：

```javascript
// root state
{
    state: () => { return { name: 'xxxx'} },
  ...,
  modules: {
     moduleA: {
        namespaced: true,
        moduleAA: {...} 
     },
     moduleB: {
       namespaced: true,
     }
  }
}
// 构造出的module结构为
// {
//  'moduleA': {
//     'moduleAA': xxx
//   },
//  'moduleB': xxx
// }
```

当然，代码实现上，为了使得寻找模块更为快速，在store上用__modulesNamespaceMap来记录{[path]: module}之间的映射关系。

以上面的demo为例子：{'moduleA': xxx, 'moduleB': xxxx, 'moduleA/moduleAA': xxx}。




##### Vuex如何构造state的呢？

1. 对rootState的监听，构造reactive的方式：

```javascript
// bind store public getters
store.getters = {}
// reset local getters cache
store._makeLocalGettersCache = Object.create(null)
const wrappedGetters = store._wrappedGetters
const computed = {}

forEachValue(wrappedGetters, (fn, key) => {
  // use computed to leverage its lazy-caching mechanism
  // direct inline function use will lead to closure preserving oldVm.
  // using partial to return function with only arguments preserved in closure environment.
  computed[key] = partial(fn, store)
  Object.defineProperty(store.getters, key, {
    get: () => store._vm[key],
    enumerable: true // for local getters
  })
})

store._vm = new Vue({
  data: {
    $$state: state
  },
  computed
})
```

在resetStoreVM实现中，可以看到store._vm是一个vue实例，而根state就维护在data下；获取根rootState就可以通过this._vm._date.$$state会去到。而对于getter的处理都归类为computed的处理了。

1. 对modules构造reactive的方式：

```javascript
  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      if (__DEV__) {
        if (moduleName in parentState) {
          console.warn(
            `[vuex] state field "${moduleName}" was overridden by a module with the same name at "${path.join('.')}"`
          )
        }
      }
      Vue.set(parentState, moduleName, module.state)
    })
  }
```

在installModule的处理过程中，对于不是根组件的state，都会通过Vue.set方式，在parentState上动态构造module.state；

通过Vue.set的方式添加对module.state变更的监听。

1. 对getters的监听

在installModule处理中，把每个getter都统一到store._wrappedGetters中：

```javascript
store._wrappedGetters[type] = function wrappedGetter (store) {
    return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters // root getters
    )
  }
```

在代码1中，我们看到其在构造vue实例的computed属性时，将所有的_wrappedGetters都挂在了computed下。



##### 对mutation的处理

同对getter的处理方式类似，所有的mutation都挂载在实例store的_mutations下了：

```
function registerMutation (store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler (payload) {
    handler.call(store, local.state, payload)
  })
}
```

##### 对action的处理

同mutation的处理类似，但是需要考虑action是异步的情况：判断处理的cb是否返回的为Promise：

```javascript
const entry = store._actions[type] || (store._actions[type] = [])
entry.push(function wrappedActionHandler (payload) {
  let res = handler.call(store, {
    dispatch: local.dispatch,
    commit: local.commit,
    getters: local.getters,
    state: local.state,
    rootGetters: store.getters,
    rootState: store.state
  }, payload)
  if (!isPromise(res)) {
    res = Promise.resolve(res)
  }
  ...
```



通过以上的处理，store的初始化已被完成，包括：

- store.state （可以获得根rootState，通过namespace path获取module.state），这其中也包括对getters的获取；
- store._mutations包含对mutation的处理；
- store._actions包含对action的处理；



#### store.commit

```javascript
this.commit = function boundCommit (type, payload, options) {
  return commit.call(store, type, payload, options)
}
```

通过上面store._mutations的封装，commit方法会通过type获取对应的mutations，在执行mutation后需要需要通知所有的subscribers。

```javascript
this._subscribers
  .slice() // shallow copy to prevent iterator invalidation if subscriber synchronously calls unsubscribe
  .forEach(sub => sub(mutation, this.state))
```



##### store.dispatch

对于action的操作要考虑其为处理异步的特性，在上面的_actions中，已经将所有的action返回结果包裹为Promise了。

![截屏2021-06-04 下午1.27.20.png](https://cdn.nlark.com/yuque/0/2021/png/388960/1622784449081-75819b56-2a9c-4102-8b4e-59c35071ba7a.png?)



##### vuex的应用

vuex同其他vue插件一样，提供了install方法，用于注入Vue及通过Vue.mixin的方式进行混入。

vuex提供的helpers方法：mapState、mapGetters、mapMutations、mapActions等，都只是通过以上收集的modules、store.getters、store.state、local context等进行分解注入。

#### 总结

总体来看，vuex的结构是非常清晰、简洁的，对于reactive的部分直接使用了Vue的能力，watch的部分也是Vue提供的能力，因此其主要的工作在于构造模块路径关系。

同redux（纯函数，真的很纯粹）相比，vuex使用了vue提供的reactive能力，其提供的nested modules（内部也进行了优化，方法快速定位某个type的module，而不是像redux那样，需要run所有的reducers获取最新的state），动态注册module也非常的方便，但这是redux并没有提供的能力。