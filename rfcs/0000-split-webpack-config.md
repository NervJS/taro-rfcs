- 提案时间: 2020-05-25
- 影响版本: 2.x / 3.x

## 概述

利用 Taro 的插件化机制来将 webpack 的部分配置（如 sass、less、uglify 等）拆分成插件，通过安装插件的方式，自由组合 webpack 配置。

## 动机

目前 Taro 的项目依赖包非常大，有些没有用到的依赖也会被安装，而这主要原因是 webpack 的配置在各个 runner 里被写死了，造成了依赖上的冗余。例如， 各个 runner 里都会默认存在 sass 相关配置，这样即使项目使用的是 less，却依然会安装 sass。所以我们希望利用插件化机制来解决这一问题，让项目需要安装的依赖更小，提升依赖安装速度与成功率。

## 使用案例

本次改进对于用户来说不会造成代码上的改动，主要影响是在依赖项上。

由于是拆分 webpack 配置，会将 webpack 配置拆分出多个修改 webpack 配置的插件，例如 `@tarojs/plugin-sass`，如果项目需要使用到 sass，则需要额外再安装这个插件，才能正常解析 sass 文件。

主要会拆分出来的插件有

- `@tarojs/plugin-sass`，向 webpack 添加 sass 相关配置
- `@tarojs/plugin-less`，向 webpack 添加 less 相关配置
- `@tarojs/plugin-stylus`，向 webpack 添加 stylus 相关配置
- `@tarojs/plugin-uglify`，向 webpack 添加 uglify 相关配置
- `@tarojs/plugin-terser`，向 webpack 添加 terser 相关配置

## 详细设计

本次改进的主要实现是基于 Taro 的 [插件化机制](https://nervjs.github.io/taro/docs/plugin) 来实现。

我们可以利用插件化机制中的 `ctx.modifyWebpackChain` API 来修改 webpack 配置，从而达到将 webpack 配置拆分出来的目的。

例如，针对 sass 的处理，我们可以实现如下插件

```typescript
// @tarojs/plugin-sass
import { IPluginContext } from '@tarojs/service'
export default (ctx: IPluginContext) => {
  ctx.modifyWebpackChain(({ chain }) => {
    chain.module
      .rule('scss')
        .test(/\.(scss|sass)$/)
        .oneOf('normal')
          .use('sass')
            .loader('sass-loader')
            .end()
          .end()
  })
}
```

安装这个插件后，webpack 才会具备处理 sass 文件的能力。

## 缺陷

- 对于新项目，我们可以把插件写到模板中，但对于旧项目升级的话会稍有麻烦，需要手动安装需要用到的插件包才能让升级后的项目运行起来，如果直接运行项目的话会报错

## 替代选择

暂无

## 适配策略

- 改进项目模板，让新创建的项目能自动安装所需插件
- 当旧项目升级后运行报错时，需要给出清晰的错误提示，引导用户安装相关插件
