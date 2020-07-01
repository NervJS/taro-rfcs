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

Vue3 与 Vue 2 的 API 存在巨大差距，只能选择重开一个新的全局框架环境变量：

```js
process.env.FRAMEWORK = 'vue3'
```

### 1. 新增 createVue3App 入口连接器

在 `@tarojs/runtime` 添加 `createVue3App` 函数，连接 Vue 3 入口组件和小程序规范的入口配置。

Vue3 页面组件与小程序页面配置的连接器可以复用现有的 `createPageConfig`。

Vue 3 所有 API 都使用 ES6 Modules 的形式导出，Taro 需要的 API 只有创建虚拟 DOM 的 `h` 函数。因此 `createVue3App` 的函数参数将变为：

```js
function createVue3App (App: App<TaroElement>, h: h, config: AppConfig)
```

createVue3App 内部可以直接使用 Vue2 的 reconciler 配置同步所有 `@tarojs/runtime` 包内的针对 Vue 的特定逻辑。

#### 使用

开发者在入口文件里只需对外导出 Vue app 实例：

```js
// app.js
import { createApp } from 'vue'

const app = createApp({
  onShow (options) {
    // ...
  }
  // 入口组件不需要实现 render 方法，即使实现了也会被 taro 所覆盖
})

export app
```

页面文件正常导出 Vue3 组件即可：

```
// page.js
<template>
  <view class="index">
    <text>{{ msg }}</text>
  </view>
</template>

<script>
import { ref } from 'vue'

export default {
  setup () {
    const msg = ref('Hello world')
    return {
      msg
    }
  }
}
</script>
```

> 相关 Vue3 RFC：


> 1. [0008-render-function-api-change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0008-render-function-api-change.md)
> 2. [0009-global-api-change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0009-global-api-change.md)

### 2. 修改 DOM 方法

* TaroNode 增加 `cloneNode` 方法
* TaroNode 允许通过 `textContent` 设置文本节点
* document 增加 `querySelector` 方法
* document 增加 `createEvent` 方法
* input 组件的 value 和当前输入值保持一致

### 3. webpack 配置改造

#### 3.1 剥离 vue-loader

之前 `mini-runner` 和 `webpack-runner` 都内置了 vue2 的 `vue-loader`，如果再内置 Vue3 的 `vue-loader` 则显得太过臃肿。因此 `mini-runner` 和 `webpack-runner` 不再内置 `vue-loader`。

现在开发者需要在项目中自行安装所需的 `vue-loader`。针对项目内找不到 `vue-loader` 的情况会进行报错与提醒，各内置模板的 package.json 也需要加上 `vue-loader` 依赖。

#### 3.2 聚合 vue3 独有的 webpack 配置

以往的 webpack 配置逻辑中常常出现以下这样的代码，目的是根据不同框架加入不同的配置：

```
if (framework === 'vue') {
  // webpack chain add something...
}
```

太多这样的条件判断会使得项目非常难以维护，可以预见到 Vue3 的兼容也需要加入一大堆这样的判断。

因此，我们可以把各框架所需的配置项聚合到各自对应的文件里，通过 **webpackChain** 修改 webpack 配置。

例如:

```js
switch (framework) {
  case FRAMEWORK_MAP.VUE:
    customVueChain(chain)
    break
  case FRAMEWORK_MAP.VUE3:
    customVue3Chain(chain)
    break
  default:
}
```

这样兼容 Vue3 只需增加一个文件，对外暴露 customVue3Chain 方法。

#### 3.3 解析 .vue 文件

当框架为 Vue3 时，也需要解析 .vue 后缀的文件。

需要修改 `mini-runner` 里的 `miniPlugin` 和 `webpack-runner` 里的 `MainPlugin` 来实现。

### 4. 组件库新增 Vue3 适配层

原本针对 Vue2，组件库提供了一层适配层进行适配。主要工作是对接由 stencil 打包 web-components，包括处理 **Vue 全局配置**、**props**、**attrs**、**ref**、**v-model**、**class** 等。

但是 Vue3 API 改动较大，故而使用 Vue3 语法重新编写提供给 Vue3 的适配层。

> 相关 Vue3 RFC：


> 1. [0008-render-function-api-change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0008-render-function-api-change.md)
> 2. [0009-global-api-change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0009-global-api-change.md)
> 3. [0011-v-model-api-change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0011-v-model-api-change.md)
> 4. [0030-emits-option](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0030-emits-option.md)
> 5. [0031-attr-fallthrough](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0031-attr-fallthrough.md)


### 5. taro-loader 改造

调用 createVue3App、对接Vue3 组件库适配层等。

### 6. Taro CLI 改造

* 创建项目时提供 vue3 框架选项，并只显示当前框架可使用的模板
* 创建项目时，跳过 **.jsx** 文件并保留 **.vue** 文件
* Joi 对 framework 字段的校验增加 `'vue3'` 选项

### 7. 其它

* 新增 Vue3 的 **default** 与 **vuex** 模板
* **eslint-config-taro** 增加 vue3 配置

## 缺陷

1. Vue 3 内部实现使用了 [Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy) ，在 iOS 9 及以下操作系统无法运行。但 Vue 官方团队在正式版发布后会推出兼容版本。
2. 在 H5 端使用 **ref** 获取基础组件的 DOM 节点，现在只能得到适配层的 Vue 组件实例，而不是对应的 webComponent 根节点。在 Vue2 里可以通过修改父元素的 refs 属性实现，但 Vue3 中组件间初始化顺序有变化，因此暂时不能支持。
3. 小程序端非类似 HTML 表单标签规范的表单组件，如 Picker，暂不兼容 **v-model**。Vue3 的 v-model 绑定属性改为了 modelValue，事件绑定改为了 update:modelValue。对于 HTML 表单标签会自动对接表单的值与事件，例如 input 会自动对应 modelValue 与 value、update:modelValue 与 @input。但对于 Picker 这种小程序特有表单则无法对应，建议这种情况不使用 v-model。
4. Vue 3 可以使用 `createRerender` 方法创建一个类似 `react-reconciler` 的渲染器，在小程序端未来可以考虑使用。但目前 Vue 3 的实现没有太多冗余浏览器兼容逻辑，可以先使用 `@vue/runtime-dom` 在小程序进行渲染。
5. VirtualList 组件需要实现一份 Vue3 版本（待实现）

## 替代选择

N/A

## 适配策略

1. 完善文档；
2. 新增 Vue 3 模板和对应 Vuex 模板;
3. 在 3.0 版本发布和后续的文章介绍/推广中有意识提及该方案的存在和用法
