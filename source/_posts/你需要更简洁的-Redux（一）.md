title: 你需要更简洁的 Redux（一）
date: 2019-09-13 17:54:38
tags:
---
使用 React 的过程中，或多或少都会接触到 **状态管理** ，从 Flux 到 Redux 到 Dva，各种状态管理工具满天飞，今天让我们来聊一聊这些工具，以及思考在这些工具的基础上如何更简洁的使用 **状态管理** 这个大杀器。

## 引子

聊状态管理之前，先让我们梳理下在原生 React 中如何进行组件间通信。

- Props

  子组件需要父组件的数据，通过 props 一层层向下传递。

  子组件需要更改父组件数据，通过 props callback 调用父组件的方法更新数据。

  多个组件共享数据，抽象 Container 组件，从 Container 组件统一分发 props。

  这样确实可以让应用的数据流非常明确，但是随着应用复杂度提升，组件层级加深，不同组件都要抽象 Container 组件，带来的状态提升问题非常严重，后期通信成本会越来越高。

- Context

  为了减少 props 带来的嵌套问题，可以使用 Context API。

  在顶级公共组件中，定义 getChildContext 方法，返回子组件所需的数据，子组件通过 this.context 访问相应数据。

  这样就省去了 props 传递带来的复杂度，但是这样也有自己的问题，比如数据过多 getChildContext 方法会非常臃肿，比如 context 跨多层组件获取时，需要上层组件都进行 render，最终组件才可获取数据等。

- 观察者模式

  观察者模式就是组件通信的一种通用方法了，在需要更新时订阅 event 的变化传入回调函数即可。

  同样的，观察者模式也会带来很多问题，比如订阅散落在各个组件中，维护成本高，违背 React 单向数据流的理念。

  所以，一个统一的状态管理工具从 React 诞生之始，就备受期待。

## Flux

Flux 其实更像是一种思想，类似 MVC、MVVM，提倡严格的单向数据流，组件所用数据全部存放在 Store 中，只能通过 Dispatcher 来告知 Store 修改数据，View 层则订阅数据变化来触发页面更新动作。

在这个思想的基础上，不管什么框架都可以基于这个实现自己的状态管理工具。

Flux 的几个主要概念：

- View：视图层，根据数据展示界面，订阅数据变化来自动更新界面，并响应用户行为触发 Action

- Store：存放和处理数据相关的逻辑

- Action：触发 Dispatcher

- Dispatcher：告知 Store 修改数据

Flux 需要手动在 view 层订阅数据变化手动触发页面更新，而且没有很好的异步解决方案，所以在 Redux 出现以后，迅速被代替。

## Redux

Redux 是在吸收 Flux 思想的基础上产生的状态管理工具。

> 此处默认 Redux 为  React-Redux

Redux 的核心概念：

- Store：管理 State，并有 dispatch、subscribe 等方法

- Action：响应用户事件，触发 Reducer

- Dispatch：分发 Action

- Reducer：修改 Store 中的数据，返回一个全新的 State

- Provider：将 store 注入到应用中

- Connect：将特定的 state 和 dispatch 函数注入到组件中

- Middleware：类似 Koa 的中间件思想，拓展 Redux 能力，如 log、dev-tool 等

虽说 Redux 吸收了 Flux 的优秀思想，但是依旧存在着众多问题，比如：

- 约定太多

  Redux 中提出了很多约定，但是根本不知道为啥要这么写，新手看起来非常头大。

  比如 Action 只是一个个 JS 对象，为什么要放在不同的文件里？

  比如 Reducer 强调纯函数，每次修改必须返回全新的 State。

  比如 Reducer 大量 case 仅仅只是为了改变一个值。

  比如应用复杂了之后，类似文件大量重复，组织繁琐。

  当然，这些可以用更好的代码组织方式来避免，不过既然是约定，如果每个人都有自己的组织方式，那约定也就失去了意义。

- 异步处理

  Redux 中的 Action 其实就是一个个 JS 对象，当需要在 Action 中进行异步操作时，只能在 Redux 上做一些封装。

  目前社区最通用的解决方案是 redux-saga，通过在全局跑着 generator，监听 dispatch 函数分发的 actionType，当命中时，进入 saga，执行异步操作，完成后再调用真实的 Reducer。

  虽说解决了很多问题，但是缺点同样明显。

  比如需要定义的文件又多了一个 saga，应用复杂后，代码组织成本直线增加。

  比如错误处理，基本每个 task 都需要 try catch 捕获错误。

  比如都 9102 年了，依旧需要使用 “丑陋” 的 yield 写法。

  综上，在直接使用 Redux 时，绝大多数都会选择对 Redux 进行一层封装，集合多种中间件的功能。

## Dva

说完 Flux 和 Redux ，发现 Dva 才是 “真香” 的解决方案。

> dva 首先是一个基于 redux 和 redux-saga 的数据流方案，然后为了简化开发体验，dva 还额外内置了 react-router 和 fetch，所以也可以理解为一个轻量级的应用框架。

以上是 Dva 的官方说明，简单理解，就是可以让 react-redux 和 redux-saga 编写的代码组织起来更合理，维护起来更简单。

比如，原先散落在各处的 Action、saga，可以在同一个对象中书写。

比如，简化配置过程，项目可快速初始化。

比如，集成 redux、redux-saga、redux-router。

比如，简洁的 API，相较于 redux 更容易理解的 app.model、app.router、app.use、app.start API。

比如，动态注册 model。

接下来又是喜闻乐见的转折，Dva 是有很多优点，但是它的缺点同样“致命”。

- Dva 是一个轻量框架

  Dva 集成了很多你可能并不需要的东西，当你想要部分使用它的功能，有点难。

  比如只能绑定使用 Dva 中的 react-router ，无法独立升级。

  比如打包脚本，想要整合进自己的项目，还需要做一番努力。

- redux-saga 的缺点

  Dva 集成 redux-saga 来实现异步 action，所以它同样继承了 saga generator 的写法，以及 saga put take 等 API，理解困难，书写不直观。

  说了这么多，最终目的还是要解决在使用 **状态管理** 时遇到的各种问题，综合对比以上的各种方案，我们简单列一下还未解决的痛点。

## 待解决的痛点

- 低入侵

  尽可能减少对原应用的入侵，提供尽量简洁的 API 来实现 Redux 功能

- 异步 Action

  提供更为现代，更为直观的异步 Action 解决方案

- 跨组件获取 Store

  子组件可以简便的获取 Stroe 树

- 模块化

  提供更自然的模块嵌套解决方案，减少心智负担

- 辅助函数

  提供 mapStateToProps、mapActionsToProps、mapReducersToProps 辅助函数，简化组件状态结构

## 总结

对比市面上流行的 **状态管理** 工具，发现并没有完全契合需求的方案，不如我们自己在 Redux 上加一层简单的封装，来解决使用过程中的痛点问题。

下一章，我们来一步步对 "Redux" 进行封装，让 "Redux" 更加简洁，更加强大。
