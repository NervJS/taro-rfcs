- 提案时间: (2021-07-27)
- 影响版本: (3.x)

## 概述

支持在开发小程序时使用 [Vue DevTools](https://devtools.vuejs.org/guide/installation.html)。

## 动机

增强使用 Vue 开发 Taro 应用的开发体验。

## 使用案例

### 1. 安装

在项目中安装 Taro 插件：

```sh
$ yarn add --dev @tarojs/plugin-vue-devtools
```

### 2. 配置 Taro 插件

在 Taro 编译配置中配置使用 `@tarojs/plugin-vue-devtools`：

```js
config = {
  plugins: [
    '@tarojs/plugin-vue-devtools'
  ],
  // ...
}
```

### 3. 编译项目

```sh
$ taro build --type weapp --watch
```

## 详细设计

### 一、Vue DevTools 实现

Vue DevTools 主要分为 **frontend** 和 **backend** 两部分，两者利用 **bridge**（socket.io）进行通讯。

- frontend：开发者工具的展示界面，是一个 Electron 应用或 Chrome 插件。
- backend：一段注入到应用运行时的代码。

### 二、在 Taro 中使用 Vue DevTools

frontend 不需要调整。

backend 有下列需要调整的地方：

#### 1. 内置手动打包的 backend

因为 NPM 分发的 `@vue/devtools` 里的 backend 代码，是使用 Webpack 以 `eval-sourcemap` 的形式打包的。但小程序不能使用 `eval`，所以我们要下载 [vuejs/devtools](https://github.com/vuejs/devtools) 仓库，修改 Webpack sourcemap 配置，然后手动打包。

```diff
// packages/build-tools/src/createConfig.js
-    devtool: 'eval-source-map',
+    devtool: false,
```

#### 2. 注入 backend 代码

首先需要把 backend 代码注入 Taro 应用并进行调用：

```js
import devtools from '@vue/devtools'

devtools.connect(
  // 'localhost',       - Host
  // undefined,         - Port
  // {
  //   io: createSocket - socket.io-client
  // }
)
```

#### 2. 模拟 `socket.io-client`

`socket.io-client` 在小程序环境不能直接使用，因此需要模拟实现 `socket.io-client` 的行为。

```js
import devtools from '@vue/devtools'

devtools.connect(
  'localhost',
  undefined,
  {
    io: function createSocket () {} // 自定义的 createSocket 方法
  }
)
```

##### 2.1 握手

需要先使用 XHR 进行[握手](https://socket.io/docs/v3/how-it-works/)通讯，获取 sessionID。

```js
{
  "sid": "FSDjX-WRwSA4zTZMALqx",
  "upgrades": ["websocket"],
  "pingInterval": 25000,
  "pingTimeout": 5000
}
```

##### 2.2 WebSocket 通讯初始化

`socket.io` 有 [upgrade](https://socket.io/docs/v3/how-it-works/) 的机制，一开始使用轮询，待 WebSocket 成功连接后切换到使用 WebSocket 通讯。

我们可以省略 upgrade 步骤，获取到 sessionID 后直接连接 WebSocket。然后进行一些初始化的通讯，待 WebSocket 真正可用后，再触发 backend 的逻辑。

WebSocket 初始化通讯：

```
<- 2probe  #ping
-> 3probe  #pong
<- 5       #upgrade
```

##### 2.3 心跳

需要定时传送心跳包，否则会被断开连接。

#### 3. 注入 DOM 方法

backend 使用了一些 Taro DOM 没有的 API，需要进行模拟。

- querySelectorAll
- element.ownerDocument
- document.defaultView
- HTMLElement

### 三、兼容 Vue2

`@vue/devtools` v6.0.0+ 是兼容 Vue2 的，但目前尤其是配合 Vuex 使用时还有部分兼容问题（如使用 Vue2 时，webpack buildin global 有问题）。

## 缺陷

- 强制锁定了 `@vue/devtools` 的版本，尤其是目前它还处于 `beta` 阶段，每次跟随升级都需要手动打包。

另外，对 devtools 功能的支持不够全面，有些功能需要针对小程序环境魔改 backend 才能实现，欢迎共建～

- 支持元素高亮。

## 替代选择

N/A

## 适配策略

- Vue 模板增加 `@tarojs/plugin-vue-devtools` 依赖。
- 后续可以增加一个参数，用于配置使用开发者项目本地安装的 `@vue/devtools`。