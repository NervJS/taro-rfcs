- 提案时间: (2022-03-29)
- 影响版本: (3.x)

## 概述

支持开发者选择使用 Webpack5 编译小程序、H5 应用。

## 动机

Webpack5 发布已有两年时间，功能足够稳定，同时其持久化缓存、模块联邦、更优的 Tree Shaking 等特性都为项目的编译流程提供了更好的解决方案。因此 Taro 将对基于 Webpack4 的编译系统进行重构，实现对 Webpack5 的支持，改善编译时的性能与体验。

## 使用案例

开发者可以自由选择编译工具。

```js
// config/index.js
const config = {
  // 使用 Webpack5 进行编译
  compiler: 'webpack5'
}
```

## 详细设计

### 1. 新增依赖包 `@tarojs/webpack5-runner`

升级 Webpack5 会带来很多差异，为了不影响目前基于 Webpack4 的项目，单独新增依赖包 `@tarojs/webpack5-runner`，并将小程序和 H5 的编译时逻辑整合在其中。

同时，旧项目需要卸载 `@tarojs/mini-runner` 和 `@tarojs/webpack-runner`，并更新 lockfile。

### 2. 更新 Plugins & Loaders

把 Taro 使用到的 Plugins 和 Loaders 升级到兼容 Webpack5 的最新版本。

| plugins | Current version | Webpack 5  support version | Latest version | Breaking Changes |
| --- | --- | --- | --- | --- |
| copy-webpack-plugin | 5.1.2 | 7.0.0 | 10.2.0 | https://github.com/webpack-contrib/copy-webpack-plugin/releases?page=1 |
| mini-css-extract-plugin | 0.9.0 | 2.0.0 | 2.4.6 | https://github.com/webpack-contrib/mini-css-extract-plugin/releases?page=2 |
| css-minimizer-webpack-plugin | - |  | 3.4.1 |  |
| terser-webpack-plugin | ^3.0.5 | 5.0.0 | 5.3.0 | https://github.com/webpack-contrib/terser-webpack-plugin/releases?page=2 |
| @pmmmwh/react-refresh-webpack-plugin | 0.4.3 | 0.5 | 0.5.4 |  |
| react-refresh | 0.9.0 |  | 0.11.0 |  |
| html-webpack-plugin | 3.2.0 |  | 5.5.0 | https://github.com/jantimon/html-webpack-plugin/blob/main/CHANGELOG.md |
| webpack-dev-server | 3.11.0 | 4 | 4.7.3 | https://github.com/webpack/webpack-dev-server/releases?page=2 |

| loader | Current version | Webpack 5  support version | Latest version | Breaking Changes |
| --- | --- | --- | --- | --- |
| css-loader | 3.4.2 | 6.0.0 | 6.5.1 | https://github.com/webpack-contrib/css-loader/releases |
| less-loader | 7.3.0 | 8.0.0 | 10.2.0 | https://github.com/webpack-contrib/less-loader/releases |
| postcss-loader | 4.3.0 | 5.0.0 | 6.2.1 | https://github.com/webpack-contrib/postcss-loader/releases |
| sass-loader | 10.2.0 | 11.0.0 | 12.4.0 | https://github.com/webpack-contrib/sass-loader/releases |
| stylus-loader | 3.0.2 | 5.0.0 | 6.2.0 | https://github.com/webpack-contrib/stylus-loader/releases |
| style-loader | 1.3.0 | 3.0.0 | 3.3.1 | https://github.com/webpack-contrib/style-loader/releases |

### 3. 废弃 `file-loader` 和 `url-loader`

使用 Webpack5 的 [asset-modules](https://webpack.js.org/guides/asset-modules/) 功能代替 `file-loader` 和 `url-loader`。兼容双方都支持的配置功能，废除在 `asset-modules` 中找不到对应的 `file-loader`、`url-loader` 配置。

### 4. 不改动 `@tarojs/loader`

Webpack 内部支持了 [this.getOptions](https://webpack.js.org/migrate/5/#getoptions-method-for-loaders)，但为了兼容 Webpack4 和维护的方便，不对 `@tarojs/loader` 进行改造。

### 5. 持久化缓存

Webpack5 的持久化缓存功能是最重要的特性之一，能极大提升再次编译时的速度。但同时也引入了**如何使缓存失效**的问题。

Taro 遵循 Webpack **“编译安全比编译速度重要”** 的理念，默认不开启持久化缓存的功能。

因此我们需要：

- 在 CLI 提醒开发者开启此功能。
- 增加 cache 编译配置方便开发者覆盖默认值。
- 提供详细文档。

#### 使缓存失效的默认策略

1. 当开发者修改 `config/index.js` 及其依赖时需要使缓存失效。
2. 当开发者切换不同的编译平台、不同的编译环境（prod / dev）时需要使缓存失效。

#### 持久化缓存的默认配置

```js
// config/index.js
const config = {
  // 使用 Webpack5 进行编译
  compiler: 'webpack5',
  cache: {
    // 是否开启持久化缓存，默认为 false
    enable: false
  }
}
```

当 `cache.enable` 为 `true` 时， Webpack 默认的 `cache` 配置为：

```js
{
  type: 'filesystem',
  // 当依赖的文件或该文件的依赖改变时，使缓存失效
  buildDependencies: {
    config: [path.join(appPath, 'config/index.js')]
  },
  // 缓存子目录的名称
  name: `${process.env.NODE_ENV}-${process.env.TARO_ENV}`
}
```


### 6. JS 压缩

新增 `jsMinimizer` 编译配置，开发者可以选择使用 `terser` 或 `esbuild` 对 JS 进行压缩。

默认使用 `terser`，因为目前两者的压缩效果还是有差距。

新增 `esbuild` 编译配置，用于配置 `esbuild` 压缩工具。

### 7. CSS 压缩

新增 `cssMinimizer` 编译配置，开发者可以选择使用 `csso`、`esbuild` 或 `@parcel/css` 对 CSS 进行压缩。

使用 `css-minimizer-webpack-plugin` 代替 `csso-webpack-plugin`，支持 `csso`、`esbuild`、`@parcel/css` 三种 CSS 压缩工具。

> [CSS Minimizer benchmark](https://goalsmashers.github.io/css-minification-benchmark/)

默认使用 `csso`，因为 `@parcel/css` 实测时容易报错，稳定性不高、而小型项目使用 `esbuild` 代替 `csso` 收益不高。

### 8. 关于接入 SWC

SWC 定位是代替 Babel，对标实现了很多 Babel 的功能，基础的语法转换应该问题不大。问题在于需要使用 SWC 插件系统把自定义的 Babel 插件实现一遍，而它的插件系统目前还难以对标 Babel 的功能（没有 path 只有节点、需要手拼 AST 节点等问题）。

SWC 对比 Babel：

* 语法编译能支持 ES5。
* 实现了对 `browserlist` 的支持，也实现了对标 `@babel/preset-env` 的 `target` 的功能。
* 实现了对标 `@babel/preset-env` 的 `useBuildIns`、`corejs` 的能力，支持 `entry`、`usage` 两种用法。但没有 `@babel/plugin-transform-runtime` 的 **core-js aliasing** 能力，也就是没有作用域隔离。
* 实现了对标 `@babel/plugin-transform-runtime` 的 **helper aliasing** 和 **regenerater aliasing** 功能，其中 `regenerator runtime` 需要宿主自行安装。
* 支持 JSX。
*  支持 React 17。
* 支持 TS。
* 支持很多 Babel 内置插件：https://swc.rs/docs/migrating-from-babel。
* 支持使用 JS 编写插件，之后会支持 Rust。


综上，目前还不能使用 SWC 代替 `babel-loader`。

Taro 的切入点：

* 使用 `@swc/register` 代替 `@babel/register`，提升加载 `config.js` 的速度。
* 使用 `@swc/jest` 代替 `ts-jest`，提升测试用例的编译速度。

### 9. 依赖预编译

受 [UmiJS mfsu](https://umijs.org/zh-CN/docs/mfsu) 特性的启发，我们可以预先把用户的 node_modules 依赖打包为一个 Module Federation 的 remote 应用，再次编译时可以免去 Webpack 对这些依赖的编译，从而提升编译速度。

依赖预编译可以分为三步：

1. 收集依赖
2. 打包依赖
3. 打包 Module Federation Remote 应用

#### 9.1. 收集依赖

参考 [Vite](https://cn.vitejs.dev/) 使用 esbuild 对用户项目进行 bundle，并编写插件分析 `import` 和 `require` 语句，收集 node_modules 依赖。

注意点：

- 对 Vue 文件的解析

#### 9.2. 打包依赖

参考 Vite 使用 esbuild 对上述收集到的依赖进行 bundle（ESM），缓存在用户项目里的 `node_modules/.taro/prebundle` 中。

注意点：

- 使用 `virtual module` 实现对输出目录的扁平化（flatten）
- 对基于 CommonJS 的依赖转换 ESM 后的编译结果进行兼容

#### 9.3. 打包 Module Federation Remote 应用

把上述 esbuild 的 bundle 打包为 Module Federation 的 remote 应用，供开发者的应用（Host）使用。

注意点：

- Webpack 的 Module Federation 加载代码并不能在小程序环境生效，需要新增 Webpack 插件魔改 `webpack/lib/container/ContainerPlugin`。

#### 9.4. 打包开发者的应用（Host），开启 Module Federation

注意点：

- 同样的 Host 端的 Module Federation 加载代码并不能在小程序环境生效，需要新增 Webpack 插件魔改 `webpack/lib/container/ContainerReferencePlugin`。

## 缺陷

- 老项目升级有一定的迁移成本
- 依赖预编译的实现复杂，需要进行广泛测试

## 替代选择

N/A

## 适配策略

- 提供详尽的升级教程与配置文档。
- 在 Taro 编译配置中尽量暴露 Webpack 配置，方便开发者进行自定义。
- 提示开发者使用开启持久化缓存、依赖预编译等优化功能。
