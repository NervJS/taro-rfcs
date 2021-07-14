- 提案时间: (2021-07-13)
- 影响版本: (3.x)

## 概述

支持在开发小程序时使用 [React DevTools](https://github.com/facebook/react/blob/main/packages/react-devtools/README.md)。

## 动机

增强使用 React 开发 Taro 应用的开发体验。

## 使用案例

### 1. 安装

在项目中安装 react-devtools（开发者自由选择版本）、Taro 插件：

```sh
$ yarn add --dev react-devtools @tarojs/plugin-react-devtools
```

### 2. 启动开发者工具

配置 npm script，如：

```json
{
  "scripts": {
    "devtools": "react-devtools"
  },
  ...
}
```

然后启动 `react-devtools`：

```sh
$ npm run devtools
```

### 3. 配置 Taro 插件

在 Taro 编译配置中配置使用 `@tarojs/plugin-react-devtools`：

```js
config = {
  plugins: [
    '@tarojs/plugin-react-devtools'
  ],
  // ...
}
```

### 4. 编译项目

```js
$ taro build --type weapp --watch --devtools
```

## 详细设计

### 一、React DevTools 实现

React DevTools 主要分为 **frontend** 和 **backend** 两部分（[OVERVIEW](https://github.com/facebook/react/blob/main/packages/react-devtools/OVERVIEW.md)）：

- frontend：开发者工具的展示界面，是一个 Electron 应用或 Chrome 插件。
- backend：一段注入到应用运行时的代码。React 渲染器（ReactDOM、React Native）会自动感知并安装 backend 代码，每当 React 修改 DOM 时，backend 会利用 **bridge**（抽象实现，如 WebSocket）通知 frontend 更新展示结果。

### 二、在 Taro 中使用 React DevTools

frontend 不需要调整，开发者安装指定版本的 `react-devtools` 即可。

backend 有下列需要调整的地方：

#### 1. 注入 backend 代码

首先需要把 backend 代码注入 Taro 应用并进行调用：

```js
const { connectToDevTools } = require('react-devtools-core/backend')

connectToDevTools({
  // host: string (defaults to "localhost") - Websocket will connect to this host.
  // port: number (defaults to 8097) - Websocket will connect to this port.
  // useHttps: boolean (defaults to false) - Websocket should use a secure protocol (wss).
  // websocket: Custom websocket to use. Overrides host and port settings if provided.
})
```

#### 2. 魔改 WebSocket

因为小程序的 WebSocket API 不遵循 Web 规范，因此需要使用小程序的 API 实现一个遵循 Web 规范的 WebSocket 对象。

```js
// 假设已实现了一个遵循 Web 规范的 WebSocket 对象
const ws = new WebSocket('ws://127.0.0.1:8097')

connectToDevTools({
  // 可以传入自定义 websocket 对象，代替 new window.WebSocket(url)
  websocket: ws
})
```

完成上述两步，frontend 和 backend 已能够相互通讯，下一步是等待 React 渲染器的接入。

#### 3. 把 React 的核心方法注入到 DevTools

React 渲染器需要调用 [react-reconciler](https://github.com/facebook/react/blob/main/packages/react-reconciler/README.md) 里导出的 `injectIntoDevTools` 方法，从而把 React 的核心方法注入到 DevTools。

如在 ReactDOM 中：

```js
var foundDevTools = injectIntoDevTools({
  findFiberByHostInstance: getClosestInstanceFromNode,
  bundleType: 1,
  version: ReactVersion,
  rendererPackageName: 'react-dom'
});
```

但 Taro 没有使用 ReactDOM，而是基于 `react-reconciler` 实现了一个自定义渲染器。因此需要在自定义渲染器中调用 `injectIntoDevTools`，如：

```js
// @tarojs/react
const TaroReconciler = Reconciler(hostConfig)
TaroReconciler.injectIntoDevTools({
  bundleType: 1,
  version: '17.0.2',
  rendererPackageName: 'taro-react'
})
```

#### 3. 处理全局变量

`react-reconciler` 里使用了全局变量 `__REACT_DEVTOOLS_GLOBAL_HOOK__`，但在小程序里这样写会报错。

因此使用 `loader` 处理 `react-reconciler` 的代码，把 `__REACT_DEVTOOLS_GLOBAL_HOOK__` 改为 `window.__REACT_DEVTOOLS_GLOBAL_HOOK__`。（`ProvidePlugin` 会自动注入 `window` 对象；没有使用 `definePlugin` 是因为和 `ProvidePlugin` 有冲突）

## 缺陷

- [为了识别 custom hooks](https://github.com/facebook/react/blob/main/packages/react-devtools/OVERVIEW.md#inspecting-hooks)，backend 会对部分符合条件的函数式组件执行 `shallow rendering`，需要注意是否存在副作用。

另外，对 devtools 功能的支持不够全面，有些功能需要针对小程序环境魔改 backend 才能实现，欢迎共建～

- 支持元素高亮。
- 在小程序的 `storage` 中记录 `filters` 变化。

## 替代选择

N/A

## 适配策略

- React 模板增加 `react-devtools` 依赖