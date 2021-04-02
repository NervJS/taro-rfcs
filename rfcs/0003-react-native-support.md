- 提案时间: 2020-09-25
- 影响版本: 3.x
- 相关 Issues: N/A

## 概述

在 Taro 中实现使用 **React** 开发 **React Native**。

## 动机

Taro 升级到 3 以后，React Native 无法使用，社区仍然有使用 Taro 开发 React Native 的需求。

## 使用案例

在页面组件和自定义组件开发中，与编写 React 项目遵循一致的[规范](https://nervjs.github.io/taro/docs/next/react/)。

入口组件生命周期、页面组件生命周期及页面事件处理函数，支持度与小程序相比会有差异，参考详细设计运行时部分。

API、组件支持度与小程序相比有些取舍，参考详细设计 API 及组件部分。

## 详细设计

本提案以 Taro 3 定义的标准为基础，完成 React Native 端的升级改造，同时在编译打包方案、API 及组件支持、React Native 项目接入灵活性等方面做了较大重构。

### 设计总结

0. 将编译打包方案统一为 [metro](https://facebook.github.io/metro/) ，更贴合 React Native 生态体系。
   1. 相比 webpack + metro 的方式，可提升编译速度，整体方案也更为清晰。
   2. 相比 webpack 多 entry 模式，可降低包大小。同时解决多 entry 模式存在的一些问题，如 [#7512](https://github.com/NervJS/taro/issues/7512)。
   3. 提供更好的 sourcemap 支持，优化开发体验。
   4. 与 React Native APP 项目自身的 metro 配置，如分包等，可灵活合并。
1. 运行时模块，按照 Taro 3 定义的标准进行改造。
   1. 与小程序及 H5 内部的写法保持基本一致，通过 metro transformer (类似 taro loader) 生成入口及页面代码，通过 taro-runtime-rn 包提供的方法包装入口组件及页面组件。
   2. 提升对页面事件处理函数的支持度。
   3. 增加对 Tab Bar 相关 API 的封装。
2. 提升 API 及组件的支持度。
   1. 按社区调研结果优先级及难易程度进行支持，仍然以 expo 体系为主。
   2. API 及组件可按需集成，对于依赖原生的 API 和组件，提供完整的原生集成文档。
   4. 视工作量情况，逐步提升支持度，同时欢迎社区贡献。
3. 提供更灵活的 React Native APP 接入方案。
   1. 不再锁定 React Native 版本，用户可在项目中自行安装 >=0.60 版本的 React Native，对于 >=0.57 && <=0.59 版本的支持将在后续推出。
   2. 除 React Navigation 外不强制依赖其他 Native Modules，用户可按需引入所需的原生依赖。相应原生依赖未安装时，相关接口不可用，但不会阻断程序运行。同时用户可根据自身应用情况，对所需 API 接口或组件进行替换。
   3. 不需要灵活定制的用户，仍可以使用我们提供的包含所需原生依赖壳工程项目，快速开始开发。

### 整体设计图示

![](https://pic6.58cdn.com.cn/nowater/fangfe/n_v2ae7a759d06224fe78ad4984928e32017.jpg)

大致流程如下：

0. `@taro/cli` 中通过`registerPlatform` 注册 `rn` 平台；
1. `yarn dev:rn` 获得编译配置，转为 `babel.config.js` 及 `metro.config.js` 配置；
2. 所有 React Native 不支持的语法及配置，通过编译配置支持；
3. 通过编译配置与 `@tarojs/taro-rn-transformer` 生成 React Native 的入口文件 `index.ts`；
4. 入口文件引入 `@tarojs/taro-runtime-rn` 使用`createReactNativeApp`进行包装；
5. 页面文件引入 `@tarojs/taro-runtime-rn` 使用`createPageConfig` 进行包装；
6. 启动 metro bunlder；
7. 在 React Native Shell 工程中运行 `react-native run-ios` 或 `react-native run-android`  加载  `index.bundle`。
### 本次新增及修改的包列表

| 包名 (packages下文件夹名)                | 包功能                                                       |
| ---------------------------------------- | ------------------------------------------------------------ |
| *taro-cli                                | 1. 获取公用编译配置及 RN 特殊配置，启动 rn-runner。<br/>2. 检查依赖包是否正常安装：包括（react-native、component-rn、runtime-rn等） |
| *taro-component-rn                       | 1. 增加对虚拟列表的支持。<br/>2. 增加对旧版本不支持的组件的支持，详见组件改造小节。 |
| *taro-rn                                 | 1. 增加对旧版本不支持的接口的支持，详见API改造小节。         |
| taro-rn-runner                           | 1. 与用户自定义的 metro 配置做合并。<br/>2. 生成babel配置。<br/>3. 引入style transformer、taro-rn-transformer 等包生成metro配置。<br/>4. 启动bundler。<br/>5. 支持通用编译配置，包括 defineConstants、copy、alias、sass、env 等。<br/>6. 支持RN特殊配置，如样式缩放开关。<br/>7. 引入 babel plugin `transform-jsx-to-stylesheet` 以支持 className 写法。<br/>8. 支持优先加载 *.rn.* 文件，Android加载 .android.文件，iOS加载 .ios.文件。<br/>9. 编译平台包替换及 alias 配置。<br/>10. 打包压缩、sourcemap 等能力支持。 |
| taro-rn-transformer                      | 1. 拦截入口文件 index.js/index.ts，根据配置，生成入口文件代码。<br/>2. 根据 app.config 的 pages 配置，拦截页面文件，根据配置，生成页面文件代码。<br/>3. 功能类似小程序及h5的 [taro-loader](https://github.com/NervJS/taro/tree/next/packages/taro-loader)。 |
| taro-style-transformer                   | 1. 支持 sass、less、stylus、postcss文件的引入，转化为 styleObject。<br/>2. 支持 sass 的全局引入[配置](https://taro-docs.jd.com/taro/docs/config-detail#sass)<br/>3. 引入 postcss-pxtransform 以支持 React Native 单位转化。 |
| taro-asset-transformer                   | 1. 支持引入静态资源，转化图片等资源为 base64。               |
| babel-plugin-transform-jsx-to-stylesheet | 1. jsx 支持 className 属性，将各种语法转化为 styleObject['className']等。 |
| bable-preset-taro-rn                     | 1. 将所需 babel plugin 进行合并，统一管理。                  |
| taro-runtime-rn                          | 1. 提供 app 包装方法。<br/>2. 提供 page 包装方法。<br/>3. 支持 pulldownrefresh 等页面事件处理函数支持。<br/>4. 支持页面配置，title、navigatebar等。<br/>5. 提供 Taro3 新增的全局对象 Current 等。 |
| taro-router-rn                           | 1. 封装所有的路由处理。<br/>2. 支持新增的TabBar相关API。<br/>3. 支持tabbar相关配置。<br/>4. 支持icon配置。 |

注：带*为做修改的包，不带为本次新增或基本重构的包。

### 编译打包方案改造

#### 对于通用配置的支持情况

| 配置            | 是否支持   | 方案                                                         |
| --------------- | ---------- | ------------------------------------------------------------ |
| sourceRoot      | 支持       | -                                                            |
| outputRoot      | 支持       | -                                                            |
| designWidth     | 支持       | -                                                            |
| defineConstants | 支持       | 使用 `babel-plugin-transform-inline-environment-variables` 加入到运行环境中。 |
| alias           | 支持       | 使用 `babel-plugin-module-resolver` 支持。                   |
| env             | 支持       | 使用 `babel-plugin-transform-inline-environment-variables` 加入到运行环境中。<br/>包含 `process.env.TARO_ENV` 值为 `rn` 。 |
| copy            | **不支持** | -                                                            |
| plugins         | **不支持** | -                                                            |
| presets         | **不支持** | -                                                            |
| terser          | 支持       | -                                                            |
| csso            | 支持       | transformer 引入 [csso](https://github.com/css/csso)。       |
| sass            | 支持       | -                                                            |

#### webpack loader 配置替代方案

通过 webpackChain 修改配置已不再支持，对样式编译配置的修改使用如下代替配置。

```javascript
rn: {
    sass: {
      options: ..., // https://github.com/sass/node-sass#options
      additionalData: ..., // {String|Function} 注入到所有 sass 文件中
      sourceMap: boolean, // 不做支持
    },
    less: {
      options: ..., // http://lesscss.org/usage/#less-options
      additionalData: ..., // {String|Function} 注入到所有 less 文件中
      sourceMap: boolean, // 不做支持
    },
    stylus: {
      options: ..., // https://stylus-lang.com/docs/js.html
      additionalData: ..., // {String|Function} 注入到所有 stylus 文件中
      sourceMap: boolean, // 不做支持
    },
    postcss: {
      options: ..., // https://github.com/postcss/postcss#options
      additionalData: ..., // {String|Function} 注入到所有 css 文件中
      sourceMap: boolean, // 不做支持
    },
 }
```

#### 平台差异化文件引用支持

通过配置 babel plugin，支持编译 React Native 平台时，优先使用 `*.rn.*`，编译 Android 平台优先使用 `*.android.*`，编译 iOS 平台优先使用 `*.ios.*`。

参考 [babel-plugin-react-native-platform-specific-extensions](https://github.com/kristerkari/babel-plugin-react-native-platform-specific-extensions) 实现。

#### 编译平台包替换及 alias 配置

将 `@tarojs/taro-components` 替换为 `@tarojs/taro-components-rn`，将 `@tarojs/taro` 替换为 `@tarojs/taro-rn`。

安装 `babel-plugin-module-resolver`，配置 `babel.config.js`。

```javascript
{
  plugins: [
    [
      'module-resolver',
      {
        alias: {
          '@tarojs/components': '@tarojs/components-rn',
          '@tarojs/taro': '@tarojs/taro-rn',
        },
      },
    ]
}
```

#### 用户 metro 配置自定义合并

taro-rn-runner 将所需的 metro 配置作为 defaultConfig，项目中的 metro.config.js 作为配置载入。

```javascript
Metro.loadConfig({}, defaultConfig);
```

#### sourcemap 支持

基于 metro 本身的 [sourcemap](https://facebook.github.io/metro/docs/cli)，可自行调整配置。

#### TypeScript 支持

[metro-react-native-babel-transformer](https://github.com/facebook/metro/tree/master/packages/metro-react-native-babel-transformer) 使用的 [metro-react-native-babel-preset](https://github.com/facebook/metro/tree/master/packages/metro-react-native-babel-preset) 默认支持 TypeScript。

### 样式语法支持

> 统一使用 transformer 实现

#### className 语法支持

通过操作 visitor 的 JSXOpeningElement，ImportDeclaration，Program.exit，实现 className 的语法转换。

参考 `babel-plugin-transform-jsx-stylesheet` 包做一些改进。

支持包括如下场景：

0. class string 与 多class string；
1. 对象 `{ active: this.props.isActive }`；
2. 数组 `['header1 header2', 'header3', { active: this.props.isActive }]`；
3. 三元运算`this.props.visible ? 'show' : 'hide'`；
4. 自定义函数 `getClassName()`；
5. 多个样式文件，对象进行合并；

#### 层叠样式表预编译语言支持（sass less stylus postcss）

通过配置 metro 的 `transformer.babelTransformerPath` 以支持 css 预编译语言，将编译结果导出为 styleObject。

参考如下实现:

0. [react-native-sass-transformer](https://github.com/kristerkari/react-native-sass-transformer)
1. [react-native-typed-postcss-transformer](https://github.com/kristerkari/react-native-typed-postcss-transformer)
2. [react-native-less-transformer](https://github.com/kristerkari/react-native-less-transformer)
3. [react-native-stylus-transformer](https://github.com/kristerkari/react-native-stylus-transformer)

#### sass.resource 支持全局

在使用 metro 的 sass transformer 时，将 `sass.resource` 配置的全局 sass 文件注入到样式文件头部。

#### 单位转化

同 2.x 通过 postcss-pxtransform 插件，在使用 metro 的 postcss transformer 时，引入该插件。该功能支持关闭。

####  样式文件中跨平台支持

如：

```
/*  #ifdef  %PLATFORM%  */
样式代码
/*  #endif  */
```

仍然通过 `postcss-pxtransform` 支持。

#### 全局的 app.css 支持

将 app.jsx 中对样式的引用，合并至页面的 styleObject 中。

#### media-query-processor 及 viewport-units 支持

参考 [react-native-dynamic-style-processor](https://github.com/kristerkari/react-native-dynamic-style-processor) 实现。

### Taro3 标准运行时支持

> 使用方法及实现方案，与小程序和h5保持一致。

#### React Navigation 升级至 5.x

> 2.x 版本的 React Navigation 已不再维护迭代。我们将路由底层库升级到最新稳定版。
> 
> 需要注意的是 5.x 版本，多了几个依赖原生的库，你的路由也将更加顺滑。

参考 [React Navigation 文档](https://reactnavigation.org/docs/getting-started)。

#### 使用 taro-rn-transformer/app.ts 生成入口文件

metro transformer 中判断是否为 index.js，是的话，使用 taro-rn-transformer/app 传入编译配置，进行转换。

transformer 伪代码如下：

```javascript
var upstreamTransformer = require("metro-react-native-babel-transformer");
const { transform } = require("@tarojs/taro-rn-transformer/app");
const { pages } = require('./config');

module.exports.transform = function({ src, filename, options }) {
  if (filename === 'index.js') {
    return upstreamTransformer.transform(transform({ src, filename, {pages, ...options}}));
  }
  return upstreamTransformer.transform({src, filename, options });
};
```

生成后入口文件的伪代码如下：

```javascript
import App from './src/App';
import {AppRegistry} from 'react-native';
import {createReactNativeApp} from '@tarojs/taro-runtime-rn';
import config from './src/App.config';

// 遍历配置生成的页面路由
import PagesIndexIndexScreen from './src/pages/index/index';
import PagesIndexAboutScreen from './src/pages/index/about';
const routers = [{
  name: 'PagesIndexIndex',
  component: PagesIndexIndexScreen
}, {
  name: 'PagesIndexAboutScreen',
  component: PagesIndexAboutScreen
}];
const buildConfig = {};

AppRegistry.registerComponent('appName', () => createReactNativeApp(App, config, routers, buildConfig));
```

再经过 metro-react-native-babel-transformer 进行二次转化。

#### 页面文件生成 taro-rn-transformer/page.ts

metro transformer 中判断是否为页面文件，是的话，使用 taro-rn-transformer/page 传入编译配置，进行转换。

生成后页面文件的伪代码如下：

```javascript
import {createPageConfig} from '@tarojs/taro-runtime-rn';
import config from './pagename.config';

// 原有的代码
import Componet from './pagename';

export default createPageConfig(Component, config);
```

再经过 metro-react-native-babel-transformer 进行二次转化。

#### createReactNativeApp

暴露给 @tarojs/taro-rn-transformer/app 调用，在 React Native 应用入口文件中调用，创建一个 React Native App 构造函数接受小程序应用规范对象。

#### createPageConfig

暴露给 @tarojs/taro-rn-transformer/page 调用，在 React Native 应用页面文件中调用，创建一个 React Native App 构造函数接受小程序页面规范对象。

#### Current

暴露给开发者的 Taro 全局变量，目前有三个属性：

0. `Current.app`，返回当前小程序应用实例，非小程序端返回小程序规范应用实例，可通过此实例调用小程序规范生命周期。
1. `Current.page`，返回当前小程序页面实例，非小程序端返回小程序规范页面实例，可通过此实例调用小程序规范生命周期。
2. `Current.router`，返回当前小程序路由信息，非小程序端返回小程序规范路由信息。

针对 React Navigation 做一样的封装。

#### 页面事件处理函数支持

| 函数              | 支持方案                              |
| ----------------- | ------------------------------------- |
| onPullDownRefresh | 基于 scrollView 的 refreshControll。  |
| onReachBottom     | 监听 onMomentumScrollEnd。            |
| onPageScroll      | 基于 scrollView。                     |
| onResize          | 基于 Dimensions。                     |
| onTabItemTap      | 基于 React Navigation tabPress 事件。 |

#### 自定义 hooks 支持

useDidShow，useDidHide，usePullDownRefresh等，参考 H5 React 实现。

#### 路由支持从外部直接带参数进入指定页面

基于 React Navigation [Deep linking](https://reactnavigation.org/docs/deep-linking/) 进行实现。

### API 改造

组件包 taro-rn fork 自 2.x 版本，增加对如下API的支持。

| API                                  | 实现方案                                        |
| ------------------------------------ | ----------------------------------------------- |
| tabBar相关API                        | 基于 React Navigation 进行封装。                |
| VideoContext                         | 基于 expo-av 进行封装。                         |
| CameraContext                        | 基于 expo-camera 进行封装。                     |
| AudioContext                         | 基于 expo-av 进行封装。                         |
| * Animation                          | 基于 Animated 进行封装。                        |
| 扫码                                 | 基于 expo-barcode-scanner 进行封装。            |
| File相关API                          | 基于 expo-file-system 进行封装                  |
| * Canvas相关API                      | 基于 react-native-canvas 封装。                 |
| 网络请求（downloadFile，uploadFile） | 基于 fetch 及 expo-file-system 进行封装。       |
| Resize 及 Keyboard 事件监听          | Resize 基于 Dimensions，Keyboard基于            |
| WebSocket                            | 基于 React Native WebSocket Support  进行封装。 |
| EventChannel，Events，eventCenter    | 基于 DeviceEventEmitter 进行封装。              |
| RequestTask相关API                   | -                                               |

注：带*为评估优先级较低。

### 组件改造

组件包 taro-component-rn fork 自 2.x 版本，增加对如下组件的支持。

| 组件                           | 实现方案                                                 |
| ------------------------------ | -------------------------------------------------------- |
| VirtualList                    | 封装 [FlatList](https://reactnative.dev/docs/flatlist)。 |
| MovableView 及 MovableArea     | 封装 Animated 及 PanResponder。                          |
| Label                          | 封装 View 当点击时，会触发对应的控件更新。               |
| PickerView 及 PickerViewColumn | 基于 @ant-design/react-native 封装。                     |
| Navigator                      | 封装对路由的操作。                                       |
| Audio                          | 基于 expo-av 进行封装。                                  |
| Video                          | 基于 expo-av 进行封装。                                  |
| Camera                         | 基于 expo-camera 进行封装。                              |
| *Canvas                        | 基于 react-native-canvas 封装。                          |

注：带*为评估优先级较低。

### APP 接入方式改造

#### appName 支持配置

因不同 APP 有不一样的 appName 配置，app.config.ts 增加配置项 appName，该参数用于注册入口组件。

```javascript
import { appName } from './app.config';
AppRegistry.registerComponent(appName, () => App);
```

#### React Native 多版本支持

一期兼容 >= 0.60 版本，初始化会默认选择最新的稳定版本，用户根据需要可自行选择可兼容的版本进行安装。同时壳工程或自身APP需要使用相应的版本。

#### 不强制依赖所有的 Native Modules

当 APP 中一些 Native Modules 不存在时，部分 API 不可用，此时弹出警告，而不是阻塞运行。

### 其他

#### 状态管理支持

同 Taro 3 [方案](https://taro-docs.jd.com/taro/docs/guide/#%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86)，由使用方自行处理。

#### SubPackages 分包

暂不支持分包，分包的配置的 subPackages 会被合并进入 pages。

#### 壳工程

保留在原工程仓库 [taro-native-shell](https://github.com/NervJS/taro-native-shell) 中，增加新的分支 0.63.0，0.62.0，0.61.0，0.60.0，均对应 Taro 3版本。

## 缺陷

0. 不支持使用 vue 或者 jQuery 框架编写 React Native。
1. 整体设计与小程序和 H5 有些差异。
2. 采用了 metro 编译打包方案，与 webpack 的方案有较大不同，需及时同步新版本迭代的更新，以支持一些新写法。

## 替代选择

N/A

## 适配策略

0. 使用 Taro React 开发的代码，对不支持的组件及接口做兼容，安装有相应依赖后，即可运行于 React Native 环境。
1. 含有部分 React Native 的原生依赖，需要增加到各自的原生仓库里，或使用壳项目进行。
2. React Native 实现的样式是 css 的子集，无法做到完全兼容，需要由业务自行兼容，做跨端项目建议 React Native first。