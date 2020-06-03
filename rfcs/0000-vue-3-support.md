- 提案时间: 2020-06-03
- 影响版本: 3.x
- 相关 Issues: N/A

## 概述

在 Taro 中实现 Vue 3 的支持，开发者可以通过 Taro CLI 构建 Vue 3 项目。

## 动机

实现 Vue 3 的支持吸引更多喜欢尝鲜追新的开发者使用 Taro。

Vue 3 将在 [2020 年 Q2](https://github.com/vuejs/vue/projects/6) 发布正式版，当前版本已经把所有 Active RFC 实现落地，不会再有 breaking-change。现在实现 Vue 3 的支持未来应该不需要进行大规模改动。

## 使用案例

在页面组件和自定义组件开发中，在 Taro 中写 Vue 3 项目就和写 Vue 2 项目遵循[一样的规范](https://nervjs.github.io/taro/docs/next/vue)，但具体的语法是由 Vue 3 和 Vue 2 之间的语法差异决定的。

在创建入口组件上具体写法略有不同：

### Vue 2

```js
import Vue from 'vue'
import './app.css'

const App = new Vue({
  // 生命周期
  onShow (options) {
  },
  render(h) {
    return h('block', this.$slots.default)
  }
})

export default App
```

### Vue 3

```js
import { createApp } from 'vue'
import './app.css'

const App = createApp({
  // 生命周期
  onShow (options) {
  }
  // Vue 3 已经支持 Fragment，不需要再套一层元素
})

export default App
```

## 详细设计

影响 Vue 3 在 Taro 中的实现主要有三个 RFC：

1. Vue RFC 0006：[slots-unification](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0006-slots-unification.md)
2. Vue RFC 0008：[render-function-api-change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0008-render-function-api-change.md)
3. Vue RFC 0009：[global-api-change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0009-global-api-change.md)

这三个 RFC 与现在 Vue 2 的实现存在巨大差距，只能选择重开一个新的全局框架环境变量：

```js
process.env.FRAMEWORK = 'vue3'
```

就具体实现而言，Taro 主要需要做的工作有 3 项：

1. 在有全局框架环境变量的地方新增 `vue3` 的判断，这部分主要涵盖的包有：`@tarojs/runtime`、`@tarojs/router`、`@tarojs/taro-h5`、`@tarojs/loader`；
2. 在 `@tarojs/runtime` 添加 `createVue3App` 函数，连接 Vue 3 页面组件和小程序规范的页面配置和路由，生命周期连接可以与当前 React/Vue2 的连接器 `createPageConfig` 公用；
3. 新增 Vue 3 模板和对应 Vuex 模板；

其中的核心工作在于实现 `createVue3App` 函数，Vue 2 的 `createVueApp` 函数接受三个函数：

```js
function createVueApp (App: VueInstance, vue: Vue, config: AppConfig)
```

但 Vue 3 所有 API 都使用 ES6 Modules 的形式导出，Taro 需要的 API 只有创建虚拟 DOM 的 `h` 函数，因此 `createVue3App` 的函数参数将变为：

```js
function createVue3App (App: App<HostElement>, h: h, config: AppConfig)
```

相应地，所有将要调用此函数的包都要随之修改，包括：`@tarojs/router`、`@tarojs/taro-loader`。

#### 注意事项

1. `createVue3App` 函数会和 `createReactApp`/`createVueApp` 一样放在 `@tarojs/runtime`，以 ES6 Modules 的形式导出供各包调用，实现时需注意避免引入冗余依赖，以免导致 tree-shaking 失效；
2. `vue-loader` 也需要更新版本，由于 `webpack/mini-runner` 现在已经将 Vue 2 版本的 `vue-loader` 内置，Vue 3 的 `vue-loader` 可以从项目目录中取;
3. 当前 Taro 包引入的 Vue Typings 为 Vue 2，使用 TypeScript 实现的包需要注意从 `@vue/runtime-dom` 引入类型，如有必要可以在 `tsconfig` 进行 `alias`

## 缺陷

1. Vue 3 内部实现使用了 [Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy) ，在 iOS 9 及以下浏览器无法运行。但 Vue 官方团队在正式版发布后会推出兼容版本。
2. Vue 3 可以使用 `createRerender` 方法创建一个类似 `react-reconciler` 的渲染器，在小程序端未来可以考虑使用。但目前 Vue 3 的实现没有太多冗余浏览器兼容逻辑，可以先使用 `@vue/runtime-dom` 在小程序进行渲染。

## 替代选择

N/A


## 适配策略

1. 完善文档；
2. 新增 Vue 3 模板和对应 Vuex 模板;
3. 在 3.0 版本发布和后续的文章介绍/推广中有意识提及该方案的存在和用法