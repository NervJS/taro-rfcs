- 提案时间: 2020-05-15
- 影响版本: 3.x
- 相关 Issues: 无

## 概述

可以在 Taro 中使用 [History API](https://developer.mozilla.org/en-US/docs/Web/API/History) 以及监听 [window popstate](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/popstate_event) 事件。

## 动机

小程序路由在使用上有两个痛点：
1. 用户需要在 `Taro.navigateTo` 和 `Taro.switchTab` 中做出选择。
2. 路由变化后没有事件通知。

如果可以实现标准化的 History API，就可以进一步使用第三方库 [history](https://github.com/ReactTraining/history)，以及以此为基础的其他路由解决方案。

## 使用案例

```jsx
window.addEventListener('popstate', (event) => {
  console.log("location: " + document.location + ", state: " + JSON.stringify(event.state));
});
history.pushState({page: 1}, "title 1", "/pages/index1");
history.pushState({page: 2}, "title 2", "/pages/index2");
history.replaceState({page: 3}, "title 3", "/pages/index3");
history.back(); // Logs "location: /pages/index1, state: {"page":1}"
history.back(); // Logs "location: /pages/index, state: null
```

## 详细设计

实现该方案“可能”需要编译时配合，记录 `app.config.js` 文件内容。

整体来说至少需要在 `@tarojs/runtime` 实现三个 API：

- `window.history`
- `window.addEventListener('popstate')` 以及 `window.onpopstate`
- `document.location`

首先需要单独维护一份页面栈信息，并且把当前页面信息同步到 `document.location`。

History API 中的几个方法和小程序是一一对应的：

| history        | Taro           |
| -------------- | -------------- |
| pushState()    | navigateTo()   |
| replaceState() | redirectTo()   |
| back()         | navigateBack() |

History API 另两个方法 `forward()` 和 `go()` 在小程序中意义不大，但是会影响到实现细节，第一阶段可以考虑暂不实现。

可以根据 `app.config.js` 中的 `tabBar.list` 字段来判断需要调用 Taro.switchTab 的地方。

在调用以上 Taro 路由方法的地方把页面信息记录在页面栈中，在不考虑 `forward()` 和 `go()` 两个方法的前提下，`navigateTo()` 入栈，`redirectTo()` 替换当前值，`navigateBack()` 出栈，`switchTab()` 重置页面栈，`<Navigator />` 组件也要做相应处理。当用户点击页面左上角“返回”按钮时，Taro 会设置 `Current.router`，此时比对页面栈信息后做出和 `navigateBack()` 一样的出栈操作。

当 `Current.router` 发生变化时，如果是后退操作，则触发 `popstate` 事件。

## 缺陷

1. History API 中的方法都是同步的，跟 Taro 中的方法在类型定义上有差异，无法兼容小程序 `events` 参数，`state` 参数在小程序中没有对应概念，`title` 参数的工作方式和现有 API 差异比较大。
2. 要实现这一目标，需要对 Taro 项目多处做改动，对未使用 History API 的用户有一定的性能影响。
3. 点击返回按钮和 `Current.router` 变化之间缺少强关联性，可能在具体实现上存在困难。
4. 单独维护的页面栈和 `getCurrentPages()` 之间存在重复，可能导致不一致。
5. Taro 从启动到真正可以调用路由方法之间存在一定的时间差，这段时间内的路由操作需要采用队列，增加了实现难度。

## 替代选择

用户可以自行封装 History API，有选择的使用 webpack ProvidePlugin 挂载在 window 对象下面。但目前没有简洁的办法监听 `Current.router` 变化。

## 适配策略

1. 在文档中把所有 API 功能/参数都详细列出来。
2. 详细说明实现上的差异。
