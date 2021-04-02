- 提案时间: 2020-07-20
- 影响版本: 3.x
- 相关 Issues: #6910

## 概述

对 Taro 为不支持模板递归的小程序（微信、QQ、京东）生成的 base 模版的体积进行优化。

## 动机

目前 Taro 对于不支持递归的小程序，会默认把代表各小程序内置组件的模板循环 16 次。

对于这种情况，我们实现了模板按需引入的优化，默认情况只会引入 `View`、`Text`、`ScrollView`、`Image` 组件，这时 base 模板大小为：*59 kb*。但是考虑到最坏情况，开发者使用了所有小程序的内置组件，这时 base 模板的体积则会爆发性增长到 *573 kb*。

小程序平台方对小程序体积的限制非常严格，开发者需要的是一个体积小巧的开发框架。因此对于 base 模板体积的优化迫在眉睫。

## 使用案例

```js
module.exports = {
  mini: {
    // 配置额外需要绑定的属性和事件
    extraTemplateAttrs: {
      view: ['hover-stop-propagation', 'bindanimationstart']
    }
    // 配置是否压缩模板
    enableCompressTemplate: true
  }
}
```

## 详细设计

优化工作可分为 Taro 对模板的重构，和新增优化工具、API 两种。前者开发者不会感知，后者需要开发者关注和配置。

### 模板重构

#### 1. 消除不必要的模板循环

在不支持模板递归的小程序中，只有这样才能做到 `view` 模板的嵌套，所以我们要把模板循环 N 次：

```html
<template name="tmpl_0_view">
  <view>
    <template is="tmpl_1_view" />
  </view>
</template>

<template name="tmpl_1_view">
  <view></view>
</template>
```

但我们发现，并不是所有组件都会嵌套调用同类组件的，例如我们不会去嵌套 Map 组件：

```html
<Map>
  <Map />
</Map>
```

因此当前我们把 `map` 模板也循环 16 次的做法就显得有些多余了，实际上 `map` 模板只需要一份即可。

具体实现上，需要引入一个 wxs 工具函数，对可嵌套组件和不可嵌套组件进行区分。

```html
<wxs module="n">
  module.exports = {
    tid: function (lv, type) {
      let name = isLoopComponent
        ? 'tmpl_' + lv + '_' + type
        : 'tmpl_0_' + type
      return name
    }
  }
</wxs>

<template name="tmpl_0_view">
  <view>
    <block wx:for="{{i.cn}}">
      <template is="tmpl_1_container" data="{{i: item}}" />
    </block>
  </view>
</template>

<template name="tmpl_0_map">
  <view>
    <block wx:for="{{i.cn}}">
      <!-- 不可嵌套的组件需要考虑如何获取当前层级 lv -->
      <template is="{{'tmpl_' + lv + '_container'}}" data="{{i: item}}" />
    </block>
  </view>
</template>

<template name="tmpl_0_container" data="{{i: i}}">
  <template is="{{n.tid(0, i.nn)}}" data="{{i: i}}" />
</template>
```


#### 2. 精简模板变量名

```html
<!-- before -->
<template name="tmpl_0_view"></template>
<template name="tmpl_0_container"></template>

<!-- after -->
<template name="t_0_view"></template>
<template name="t_0"></template>
```

### 优化工具

#### 1. 精简属性

因为小程序组件的属性和事件都必须静态写死，不可以动态添加，所以 Taro 会把组件的所有属性和事件全部在模板里提前进行绑定。

但实际项目中很多情况下并不会使用到组件的所有属性和事件，循环这些冗余的属性和事件绑定也会占据很大一部分的体积，另外太多的事件绑定也会在一定程度上降低小程序的性能。

因此我们可以借助 AST 分析用户使用到的组件属性和事件绑定：类 React 框架使用 webpack 的 parser 识别 **JSXAttribute**、Vue 可以借助 `vue-loader` 的 compilerOptions、jQuery Like 可以识别 `.attr` 和 `.on` 的调用。然后只在模板上只添加这些属性和事件。

但是静态分析有自身的局限性，有几种情况是分析不到的：

* JSXSpreadAttribute
* React.createElement 方法中传入的属性
* Vue JSX 语法中绑定的属性和事件
* Vue3 attribute fallthrough

针对这些情况，可以提供一个 `taro mini` 的配置项，对组件需要的属性和事件进行指定。

考虑到对上述配置项的维护成本和开发时的心智负担都非常重，此项优化默认不开启，可以作为一个额外的选项提供给开发者，在体积要求非常苛刻的场景下才推荐使用。

#### 2. 模板压缩

在生产环境默认使用 `html-minifier-terser` 对 base 模板进行压缩。

## 关于分包

Issue 上有不少同学建议把 base 模板放到分包。但是所有页面的渲染都依赖于 base 模板，把它放到分包是不可行的。

为分包页面单独生成一份 base 模板并放到分包内在技术上是可行的，但对体积的优化不大，只有当分包页使用了许多主包没使用的组件时，才会有一定收益。

## 缺陷

- 日后支持更多 Web 开发框架时，**精简属性功能**都要针对框架的语法进行适配
- **精简属性功能**对开发者来说维护、迁移成本和开发时的心智负担都很重

## 替代选择

N/A

## 适配策略

1. 对模板的重构工作改动很多，需要先进行详尽的测试，尽量覆盖全场景，再发布版本。
2. 对 **精简属性功能**、**模板压缩功能** 新增的配置项在文档中进行详细说明。针对前者会影响开发者代码表现的配置，默认不开启。