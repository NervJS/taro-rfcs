- 提案时间: 2021-6-28
- 影响版本: 3.x

## 概述

在 Taro 中实现对 HarmonyOS 的支持，开发者可以使用 Taro 开发鸿蒙应用。

## 动机

因为鸿蒙应用支持使用 JS 开发 UI 层，其语法与小程序相接近，所以我们希望把 Taro 多端适配的能力扩展到鸿蒙平台上，达成三大目标：

* 开发者可以使用 Taro 开发鸿蒙应用。
* 开发者可以把现有的 Taro 应用适配到鸿蒙平台。
* 开发者可以使用 Taro 的反向转换工具，把原生开发的小程序转换为 Taro 应用，再适配到鸿蒙平台。

## 使用案例

### 1. 配置插件

```js
// config/index.js
config = {
  // ...
  plugins: ['@tarojs/plugin-platform-harmonyos']
}
```

### 2. 使用编译命令

```bash
# dev
taro build --type harmonyos --watch
# prod
taro build --type harmonyos
```

## 详细设计

鸿蒙 JS UI 的开发方式与小程序类似，因此可以很大程度地沿用 Taro 在小程序端的架构。

> Taro **小程序端**架构的渲染流程图：

![](http://storage.360buyimg.com/cjj-pub-images/taro-weapp-s.png)

接下来将简单介绍实现方案。

### 一、插件化

Taro 将以 [端平台插件](https://docs.taro.zone/docs/platform-plugin) 的形式支持 HarmonyOS 平台，主要的兼容工作由此插件完成。

开发者引入插件 `@tarojs/plugin-platform-harmonyos` 即可编译为鸿蒙应用。

### 二、 渲染方案

基本复用小程序端的渲染方案，若干部分需要进行微调：

#### 1. 模板

在小程序端，Taro 使用 `<template>` 作为 DOM 节点进行渲染。而鸿蒙目前还不支持 `<template>`，只能使用**自定义组件**进行代替。但自定义组件的开销大，节点过多时会出现**性能问题**。

因此，需要使用端平台插件的[修改模板功能](https://docs.taro.zone/docs/platform-plugin-template)，对生成的模板内容进行改造。

#### 2. 运行时

> @待定 （v3.3 上线后确定）
> 目前有两种方法对运行时 @tarojs/runtime 包内容进行修改：
> 1. 直接拷贝一份 @tarojs/runtime 到鸿蒙插件中（快捷但难以维护）
> 2. @tarojs/runtime 使用 @rollup/plugin-alias 根据环境变量打包鸿蒙版本的 runtime

##### 2.1 数据绑定

使用 `this.$set` 代替 `this.setData`。

##### 2.2 生命周期

修改 `App` 和 `Page` 构造对象的生命周期为鸿蒙规范。

##### 2.3 路由参数

鸿蒙生命周期函数中获取不到路由参数，需要从 `this.data` 中获取。

##### 2.4 事件

把事件对象的类型 `click` 改为 `tap`，兼容小程序规范。
	
### 三、 组件库

> 工作量大，需要和社区进行共建

使用鸿蒙原生语法去封装一系列符合微信小程序规范的自定义组件。

如使用鸿蒙的 `<input>` 组件去封装一个遵循微信小程序规范的 `<checkbox>` 自定义组件。

### 四、  API

> 工作量大，需要和社区进行共建

使用鸿蒙的 API 去封装遵循微信小程序规范的 API。

如使用鸿蒙 `@system.router` 提供的路由能力去实现微信小程序规范中的 `navigateTo`、`redirectTo` 等 API。

###  五、样式
	
鸿蒙只支持部分的 CSS 样式，开发者需要根据鸿蒙的规范手动进行调整。

鸿蒙 CSS 样式参考：[文档](https://developer.harmonyos.com/cn/docs/documentation/doc-references/js-framework-syntax-css-0000000000611425)

### 六、 调整 App、Page、Component 的构造方式

在小程序中，App、Page、自定义组件都有各自的构造器：

```js
// 入口构造器
App({})
// 页面构造器
Page({})
// 自定义组件构造器
Component({})
```

而在鸿蒙中，则都统一为导出一个默认对象：

```js
export default {}
```

因此需要把 Webpack 的编译产物修改为以下形式：

```js
var taroExport;

(
  // webpack code...
  taroExport = TheTaroMethod()
  // webpack code...
);

export default taroExport;
```

### 七、编译调整

> @待定

鸿蒙应用可以由多个 PA 组成，每个 PA 里又可以包括多个 JS FA（对应多个 JS 项目）。

##### 多 PA

以 monorepo 的形式组织项目，共享依赖和公共逻辑。

##### 多 FA

以往每个 Taro 项目对应编译出一个小程序项目（对应鸿蒙的一个 JS FA），因此 Taro 需要支持多项目编译。

## 缺陷

* 目前使用自定义组件作为 `<template>` 的替代进行渲染，存在性能问题。
* 目前使用 `globalThis` 实现全局对象，不清楚鸿蒙将来是否会修改此 API。
* 鸿蒙手机版本不支持事件冒泡，而 Taro 需要把事件提前绑定好，因此会存在事件触发不了的问题。
* 页面跳转后，所有事件都不触发，待查明。
* 现有项目适配鸿蒙需要调整样式兼容性。

#### 鸿蒙存在的问题

* ES Module 实现有误，导出的模块没有按 ES 规范保持对数据的引用。
* 手机版本不支持事件冒泡。
* 自定义组件不能继承页面样式。
* 不兼容部分样式属性。

## 替代选择

N/A

## 适配策略

1. 完善文档；
2. 推出适配鸿蒙的 Demo；
