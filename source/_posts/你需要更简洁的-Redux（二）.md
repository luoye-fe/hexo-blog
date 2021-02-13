title: 你需要更简洁的 Redux（二）
date: 2019-09-18 12:44:01
tags:
---

## 上期回顾

首先我们先来回顾一下上一篇文章我们总结的几个痛点：

- 低入侵

- 异步 Action

- 跨组件获取 Store

- 模块化

- 辅助函数

然后我们针对以上痛点，一步步实现对 Redux 的封装。

## API 设计

在开始编码之前，容我们先花一点时间按照痛点来定一下 API 应该是什么样子的。

首先，Redux 的注入应该是非常简单的，不入侵我们业务代码的，类似 Dva 那种彻底改变应用启动方式的方式不是首选方案。所以，采用修饰器是相对来说比较稳妥的方案，不用在原来业务代码上做太多的修改。

```javascript
// 根组件注入 Redux
@reduxRoot({ store })
class Index extends PureComponent {
  // xxx
}
// 子组件注入 Redux （为了子组件可以直接获取整颗 store 树，并提供辅助函数）
@reduxChild()
class Child extends PureComponent {
    // xxx
}
```

再者，我们还需优化原生 Redux 将 Reducer、Action 等概念散落在不同文件且区分不明确的方式。

```javascript
// 用 object 组织 store 树
export default {
  state: {}, // 状态集合
  reducers: { // 真正修改 state 的同步函数集合
    ADD(state, payload) {
      // xxx
    }
  },
  actions: { // 触发 reducers 的函数集合，支持异步
    add({ dispatch, commit, state }, payload) {
      dispatch(/* xxx */)
      // or
      commit(/* xxx */)
    },
    asyncAdd({ dispatch, commit, state }, payload) {
      setTimeout(() => {
        dispatch(/* xxx */)
      }, 1000)
    },
  },
  modules: {}, // 同样结构嵌套的模块集合
}

```
最后，需要统一一下调用 Action 和 Reducer 的方式，为了区分这两种，Action 用 dispatch 方法触发，Reducer 用 commit 方法触发，两者调用方式类似

```javascript
// 调用 Action
this.props.dispatch('add', 1, state => {})
this.props.dispatch('add', 1).then(state => {})
this.props.dispatch('asyncAdd', 1).then(state => {})
this.props.dispatch({ type: 'asyncAdd', payload: 1 }).then(state => {})
this.props.dispatch('moduleA/add', 1, state => {}) // 模块化 提交
// 调用 Reducer
this.props.commit('ADD', 1)
this.props.commit({ type: 'ADD', payload: 1 })
this.props.commit('moduleA/ADD', 1) // 模块化 提交
```

接下来我们来看代码实现

## 嵌套模块化的实现

```javascript
export default {
  state: {},
  reducers: {},
  actions: {},
  modules: {},
}
```

当 modules 里嵌套多层的时候，我们可以将整颗 store 树当做一个叉树来看，需要完成整合所有state，根据路径获取 module 中的值，根据路径更新某 module 的 state 等功能。

```javascript
export default class Module {
  constructor(rawModule) {
    this.state = null
    this._rawModule = rawModule
    this.collectionState([], rawModule)
  }
  // 分割路径
  getPath(pathString) {
    return pathString.split('/').slice(0, -1)
  }
  // 根据路径获取具体 state
  getState(path) {
    return path.reduce((state, key) => state[key], this.state)
  }
  // 根据路径获取具体 rawModule
  getRawModule(path) {
    return path.reduce((module, key) => {
      if (!module.modules) return null
      return module.modules[key]
    }, this._rawModule)
  }
  // 获取 module 中的 actions 列表
  getActions(module) {
    return {
      ...(module.actions || {}),
    }
  }
  // 更新整棵树
  updateState(path, nextState) {
    if (path.length === 0) return this.state = nextState
    return chainDefine(this.state, path, nextState)
  }
  // 递归整颗树，获取 state 集合
  collectionState(path, rawModule) {
    if (path.length === 0) {
      this.state = rawModule.state
    } else {
      const parent = this.getState(path.slice(0, -1))
      parent[path[path.length - 1]] = rawModule.state
    }
    if (rawModule.modules) {
      objForEach(rawModule.modules, (rawChildModule, key) => {
        this.collectionState(path.concat(key), rawChildModule)
      })
    }
  }
}
// 遍历对象
export function objForEach(obj, callback) {
  Object.keys(obj).forEach(key => callback(obj[key], key))
}
// 链式赋值
export function chainDefine(map, path, value) {
  if (path.length === 1) return map[path[0]] = value
  return chainDefine(map[path[0]], path.slice(1), value)
}
```

## ReduxRoot + ReduxChild

完成了基础类再来完成最关键的 Redux 注入部分

```javascript

import { createStore } from 'redux'
// 为根组件注入 Redux
function reduxRoot({
  store,
}) {
  return OriginComponent => {
    const reactReduxStore = createStore(() => {
      // 实现处理不同 reducer 的逻辑
    })

    class RootStoreComponent extends PureComponent {
      getChildContext() {
        return {
          commit: this.commit,
          dispatch: this.dispatch,
          originStore: reactReduxStore, // Redux 原始对象，可调用 getState 等方法
        }
      }

      commit() {
        // 用于提交 Reducers
      }

      dispatch() {
        // 用于提交 Actions
      }

      render() {
        return (
          <OriginComponent />
        )
      }
    }

    const ConnectComponent = connect(state => ({ state }))(RootStoreComponent)

    class ProviderComponent extends PureComponent {
      render() {
        return (
          <Provider store={reactReduxStore}>
            <ConnectComponent />
          </Provider>
        )
      }
    }

    return ProviderComponent
  }
}
// 为子组件注入 Redux
function reduxChild(options = {}) {
  return OriginComponent => {
    class ChildStoreComponent extends PureComponent {
      static contextTypes = {
        commit: PropTypes.func,
        dispatch: PropTypes.func,
        originStore: PropTypes.object,
      }
      render() {
        return (
          <OriginComponent {...this.props} />
        )
      }
    }
    return connect(state => ({ state }))(ChildStoreComponent)
  }
}
```

整体上的思路就是通过装饰器，在根组件上使用 Provider组件 和 connect方法，初始化 Redux，在子组件上通过 context 将跟组件的 commit、dispatch 等方法挂载到子组件，并通过 connect 将 state 注入到子组件中。

## Dispatch、Commit 及 异步 Action

其实异步 Action 的实现相对简单，因为在原生 Redux 中 Action 更像是一个常量，如果我们将 Action 看做一个函数，在 Action 函数执行完毕后，再真正调用 this.props.dispatch 触发原生 reducer 即可实现异步 Action。

```javascript
// ...
commit() {
  const formattedArguments = normalizeArguments(args)
  // 此处的 dispatch 为 原生 Redux 的方法，提交后，全部走到 createStore 的回调函数中处理相关逻辑
  this.props.dispatch({
    type: formattedArguments.type,
    payload: formattedArguments.payload,
  })
}
dispatch(...args) {
  return new Promise(resolve => {
    // 格式化参数，支持载荷和对象两种提交方式
    const formattedArguments = normalizeArguments(args)
    const type = formattedArguments.type
    const payload = formattedArguments.payload
    const callback = formattedArguments.callback
    // 等待 action 执行完成再 向 redux dispatch 真实 action，实现异步 action
    const splitArray = type.split('/')
    const actionFuncName = splitArray[splitArray.length - 1]
    const path = StoreModules.getPath(type)
    const rawModule = StoreModules.getRawModule(path)

    function dispatchHelper(...argsArr) {
      // argsArr  ['reducerName', payload]
      // 子模块提交需要修正 reducerName
      let fixedReducerName = argsArr[0]
      if (splitArray.length > 1) fixedReducerName = splitArray.slice(0, splitArray.length - 1).concat([fixedReducerName]).join('/')

      // 触发真正的 reducer
      this.commit(fixedReducerName, argsArr[1])
      resolve(this.getState())
      callback(this.getState())
    }

    const actionsCollect = StoreModules.getActions(rawModule)

    // 将相应参数传入到 Action 函数中以便调用
    return actionsCollect[actionFuncName]({
      commit: dispatchHelper.bind(this), // 在 Action 中执行完异步操作，触发 commit 函数，才会触发 store 的更新
      state: StoreModules.getState(path),
      dispatch: this.dispatch,
    }, payload)
  })
}
// ...
// 工具函数
// 将参数格式化成统一格式
function normalizeArguments(args) {
  let type
  let payload
  let callback
  if (typeof args[0] === 'object') {
    type = args[0].type
    payload = args[0].payload
    callback = args[1] || function () {}
  } else {
    type = args[0]
    if (typeof args[1] === 'function') {
      callback = args[1]
    } else {
      payload = args[1]
      callback = args[2] || function () {}
    }
  }
  return {
    type,
    payload,
    callback,
  }
}
```

dispatch 和 commit 的主要思路是 dispatch 用于执行 store 中定义的 action 函数，在 action 中可以用 commit 方法触发原生的 reducer，在原生 reducer 中触发 store 中定义的各种 reducer 函数，来达到更新 state 的目的，接下来我们看 createStore 中的回调函数如何实现分发 reducer。

```javascript
import { createStore, applyMiddleware, compose } from 'redux'
// 支持 Redux DevTools
const composeEnhancers = window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ || compose
const reactReduxStore = createStore((...args) => {
  const subscriber = args[1]

  // 过滤 redux 的初始化事件
  if (/@@/.test(subscriber.type)) return StoreModules.state
  const path = StoreModules.getPath(subscriber.type)
  const splitArray = subscriber.type.split('/')
  const reducerFuncName = splitArray[splitArray.length - 1]
  const rawModule = StoreModules.getRawModule(path)
  const rawModuleState = StoreModules.getState(path)
  if (!rawModule) return StoreModules.state
  if (!rawModule.reducers || !rawModule.reducers[reducerFuncName]) {
     // reducer func 不存在
    return StoreModules.state
  }

  // 调用 store 中定义的 reducer 函数，来获取新的 state
  const nextState = rawModule.reducers[reducerFuncName](rawModuleState, subscriber.payload)
  // 以下一个状态更新当前 state 树
  StoreModules.updateState(path, nextState)
  // 返回全新的 state，来达到更新的目的
  return {
    ...StoreModules.state,
  }

  // 处理 Redux 中间件逻辑，在 reduxRoot 初始化是可注入多个 middlewares
}, composeEnhancers(applyMiddleware(...middlewares)))
// xxx
```

到这里，我们上面提到的痛点大部分都已解决，以前需要理解多个概念，多写很多代码的 “学院派” Redux，已经离我们而去，剩下的是代码一看即懂的 “简洁版” Redux。

还能更方便更简洁嘛？当然。

## 辅助函数

在真实的业务场景中，我们并不用将所有的 state/action/reducer 都注入到组件的 props 上，所以我们可以实现 mapStateToProps、mapActionsToProps、mapReducersToProps 这样的辅助函数来简化我们的组件属性。

```javascript
// 辅助函数的调用
@reduxRoot({
  store,
  mapStateToProps: { // 可以对 state 中的值进行二次计算再返回
    index: state => state.index,
    childIndex: state => state.child.index, // 子 module
  },
  mapReducersToProps: ['ADD', 'child/ADD'],
  mapActionsToProps: ['asyncAdd', 'add', 'child/add', 'child/asyncAdd'],
})
class Index extends PureComponent {
  // xxx
}
// 或者
@reduxRoot({
  store,
  mapStateToProps(state) {
    return {
      index: state.index
    }
  },
  mapReducersToProps: {
    SELF_ADD: 'ADD'
  },
  mapActionsToProps: {
    selfAdd: 'add'
  }
})
class Index extends PureComponent {
  // xxx
}
```

不管是对象形式还是函数形式，最终都是会返回一个对象，在 render 过程中，可以将三个辅助函数的计算结果通过解构传入到组件中，以便组件中直接使用。

```javascript
// xxx
render() {
  const mapStateResult = mapStateToPropsHelper(mapStateToProps, this.getState())
  const mapActionsResult = mapActionsToPropsHelper(mapActionsToProps, this.dispatch)
  const mapReducersResult = mapReducersToPropsHelper(mapReducersToProps, this.commit)
  return (
    <OriginComponent {...this.props}
      {...mapStateResult}
      {...mapActionsResult}
      {...mapReducersResult}
      dispatch={this.dispatch}
      commit={this.commit}
      store={{
        state: this.getState(),
        dispatch: this.dispatch,
        commit: this.commit,
        originStore: reactReduxStore,
      }}
    />
  )
}
// xxx
// 子组件同理，不再赘述
```

至此，一个 “简洁版” Redux 已经可以在生产环境使用，当然，还有一些边界情况没有在伪代码中反映出来，比如 Action、Reducer 不存在时的提示等。
