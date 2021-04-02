- 提案时间: (格式为2021-03-18)
- 影响版本: (3.x)

## 概述

支持使用 H5 标签编写 Taro 应用。

## 动机

多年以来 Web 端沉淀了大量优秀的库和组件，希望能直接在小程序端进行复用，解放生成力。

希望能减少 H5 应用迁移到小程序端的成本，甚至能够直接运行在小程序端。

## 使用案例

### 配置插件

```js
// config/index.js
config = {
  // ...
  plugins: ['@tarojs/plugin-html']
}
```

### React

```jsx
function Index () {
  return (
    <div>Hello World!</div>
  )
}
```

### Vue

```html
<template>
  <div>Hello World!</div>
</template>
```

### 引用 Antd

```jsx
import { Button } from 'antd-mobile'

function Index () {
  return (
    <Button>default</Button>
  )
}
```

## 详细设计

### 插件化

项目体积对于小程序来说是很重要的，因此我们选择以 `Taro 插件` 的形式引入此功能，开发者可以按需要进行引入。

### HTML 标签的渲染方案

开始之前，我们认为有三种可以渲染 HTML 标签的方案：

#### 1. 自定义组件

优点：兼容性强。实现简单。支持 CSS 标签名选择器。
缺点：性能问题。增加体积。

#### 2. 自定义组件（Virtual Node）

优点：实现简单。支持 CSS 标签名选择器。
缺点：兼容性弱（目前只有微信、京东小程序支持）。增加体积。

#### 3. `<template>`

优点：兼容性强。
缺点：不支持标签名选择器（有解决办法）。

综上，**性能**和**兼容性**是最关键的，其它问题都有办法绕过。

针对性能，我们使用上述三种方案，在一个页面中渲染 4000 个节点时，对比如下：

`<template>` **1.1s** < 自定义组件（Virtual Node）**1.4s** < 自定义组件 **1.9s**

可见 `<template>` 在性能和兼容性上都优于自定义组件，所以还是沿用目前的 `<template>` 进行渲染。

### HTML 标签与小程序组件的对应

Taro3 的渲染路径如下：

**前端框架 -> Taro DOM -> 小程序 data** 

HTML 标签和小程序组件的标签名、属性、事件是有差异的，而前端框架无需感知这些差异。

因此 Taro DOM 层不需要改动，只要在 **Taro DOM 序列化为小程序 data** 这一步作影射，也就是在 `hydrate` 步骤修改 Taro DOM 序列化的数据结果即可。

#### 标签名影射

当 Taro DOM 序列化数据的 `nn` 字段为 HTML 标签时，影射为对应的小程序组件名称。（具体影射规则请看附录一）

##### 表单组件

HTML 标签和小程序组件两种规范中，存在较大差异的部分主要是表单组件。

1. 在使用 `<input type='checkbox'>` 或 `<input type='radio'>` 时，需要手动补充 `<CheckboxGroup>`、`<RadioGroup>` 组件。
2. HTML 使用 `<select>` + `<option>` 实现选择器，而小程序使用 `<picker>`。两者差异巨大，不作影射。当用户使用了 `<select>` 时，提示用户直接使用 `<Picker>` 组件。

```js
// Taro DOM 的序列化数据
{
  nn: 'img'
}
// 影射结果
{
  nn: 'image'
}
```

#### 属性影射

当 Taro DOM 序列化数据的 `nn` 字段为 HTML 标签时，对组件的属性或属性值作对应。（具体影射规则请看附录二）

```js
// Taro DOM 的序列化数据
{
  nn: 'a',
  href: 'xxx'
  target: '_blank'
}
// 影射结果
{
  nn: 'navigator',
  url: 'xxx',
  openType: 'navigate'
}
```

#### 事件影射

需要对 HTML 标签和小程序组件间有差异的事件名进行影射：

| HTML 事件 | 小程序组件事件 |
| :------- | :----------- |
|  click   |      tap     |

### 样式

#### 浏览器默认样式

Taro 提供两种内置的浏览器默认样式，可以直接引入生效：

- `@tarojs/taro/html.css`: W3C HTML4 的内置样式，只有 HTML4 标签样式，体积较小，兼容性强，能适应大多数情况。
- `@tarojs/taro/html5.css`: Chrome(Blink) HTML5 的内置样式，内置样式丰富，包括了大多数 HTML5 标签，体积较大，不一定支持所有小程序容器。

#### 兼容 CSS 标签选择器

在 `template` 方案中，真正渲染的节点都是小程序组件。使用标签选择器（如：`div {}`）是不能正常工作的。

因此，可以使用类名去进行模拟：

1. 所有 H5 标签都加上类名： `h5-${tagName}`。
2. 使用 postcss 插件处理标签名选择器：

```css
// 标签名选择器
div {}
// 经 postcss 插件处理后变为类名选择器
.h5-div {}
```

#### 通配符 *

在小程序中使用 `*` 通配符会报错。

而把 `*` 替换为所有 HTML 类名选择器虽然可以实现“通配”的目的，但会引起选择器权重问题。

因此，目前 postcss 插件会删除包含 `*` 选择器的样式块。

#### H5 组件库的尺寸变小一倍

Taro 默认会启动 postcss 插件 `pxtransform`，按比例**把 px 解析为 rpx**。

但 H5 组件库不需要被此插件解析，用户可以配置 `pxtransform` 插件进行过滤：

```js
// config/index.js
config = {
  mini: {
    postcss: {
      pxtransform: {
        enable: true,
        config: {
          selectorBlackList: [/weui/]
        }
      }
    }
  }
}
```

#### rem -> rpx

待支持

### 限制

H5 标准和小程序标准存在着很大的差异，有一些 Taro 能够抹平，但仍有部分差异无法处理：

#### 1. `<img>` 图片自适应宽度

在 H5 中，不规定 `<img>` 的宽高时，浏览器会使用原图的宽高作标签的宽高。

而在小程序中，有默认的内置样式设置 `<image>` 标签的尺寸。

解决办法：用户在使用 `<img>` 时必须显式设置它的宽高。

#### 2. 部分 H5 的样式或 CSS 选择器，小程序不支持

不处理

#### 3. DOM API 差异

`canvas`、`video`、`audio` 等组件在 H5 端可以直接调用 `HTMLElement` 上的方法。

```js
// h5
const el = document.getElementById('myVideo')
el.play()
```

但是在 Taro 中，要调用组件上的原生方法，必须先创建对应的 `Context`：

```js
// 小程序
const ctx = Taro.createVideoContext('myVideo')
ctx.play()
```

## 缺陷

- 小程序中获取 DOM 尺寸的 API 是异步的，因此无法使用需要获取 DOM 尺寸（如：`el.offsetWidth`）的库。
- Taro 使用 React Reconciler 实现了自定义的渲染器，因此部分基于 ReactDOM 实现的 H5 组件会无法使用，如：`unstable_renderSubtreeIntoContainer`。
- Taro Runtime Reconciler 的 hooks 目前是一对一，应该改造成一对多，否则会有冲突问题。
- 因为是直接修改 Taro DOM 树的序列化数据，可能会令不同小程序、不同框架的数据处理逻辑耦合。

## 替代选择

N/A

## 适配策略

1. 完善文档；
2. 新增使用 H5 标签开发的项目模板和配合使用 Antd、ElementUI 组件库的模板；
3. 新增 Taro 配合使用各种 Web 生态库的示例项目，如 WeUI、Antd、ElementUI；
4. 把此功能需要配置的选项归一到插件选项中，让开发者只关注 `@tarojs/plugin-html` 插件，如 postcss 配置。

## Todos

- vue 兼容
- rem -> rpx

## 附录

### 一、标签名影射

| HTML 标签 | 小程序组件 |
| :------- | :-------- |
|  块级标签  |   view   |
|  内联标签  |   text   |
|   img    |   image  |
|     a    | navigator |
|a[\"href=javascript;\"]| view |
|  input   |  input   |
|input[\"type=checkbox\"]| checkbox |
|input[\"type=radio\"]| radio |
|  button  |  button  |
| textarea | textarea |
| progress | progress |
|  label   |  label   |
|   form   |   form   |
|  audio   |  audio   |
|  canvas  |  canvas  |
|  video   |  video   |
|  iframe  | web-view |
|   slot   |   slot   |

### 二、属性名影射

#### `<a>`

| `<a>` 属性 | `<navigator>` 属性 |
| :----------- | :----------- |
|     href     |      url     |
|    target    |   openType   |

`target` 属性值：

| `a[target]` | `navigator[open-type]` |
| :----------- | :----------- |
|    _blank    |   navigate   |
|    _self     |   redirect   |

#### `<input>`

| `<input>` 属性 | `<input>` 属性 |
| :----------- | :----------- |
|  autofocus   |    focus     |
|   readonly   |   disabled   |

`type` 属性值：

| `input[type]` | `input[type]` |
| :----------- | :----------- |
|    tel    |   number   |
| `input[type=password]` | `input[password=true]` |

#### `<textarea>`

| `<textarea>` 属性 | `<textarea>` 属性 |
| :----------- | :----------- |
|  autofocus   |    focus     |
|   readonly   |   disabled   |

#### `<progress>`

| `<progress>` 属性 | `<progress>` 属性 |
| :----------- | :----------- |
|   value / max * 100   |   precent   |

#### `<button>`

| `button[type]` | `button[form-type]` |
| :----------- | :----------- |
| `button[type=submit]` | `button[form-type=submit]` |
| `button[type=reset]`  | `button[form-type=reset]` |
