- 提案时间: (2023-10-27)
- 影响版本: (3.x)
- 相关 issues: [#7400](https://github.com/NervJS/taro/issues/7400)

## 概述

**小程序编译模式（CompileMode）**通过在编译阶段对开发者的代码进行扫描，把 JSX、Vue template 代码提前编译为对应的小程序模板代码，达到减少小程序渲染层虚拟 DOM 树节点数量的效果，从而提升渲染性能。

## 动机

Taro3 是重运行时框架，当节点数量增多到一定量级时，渲染性能会大幅下降，出现白屏时间长、交互延时等问题。

之前在开发 [CustomWrapper](https://docs.taro.zone/docs/next/optimized#2-customwrapper-%E7%BB%84%E4%BB%B6) 时，我们发现页面局部的更新使用对应自定义组件的 `setData` 会比使用页面的 `setData` 性能要更好。这是因为使用后者时小程序渲染层会 diff 整颗页面的虚拟 DOM 树，而前者则只会 diff 自定义组件范围内的虚拟 DOM 子树。可见在节点数过多时，小程序渲染层的 diff 操作可能会消耗大量耗时。

在分析微信小程序的模板解析算法后，我们发现以下模板语法会额外产生虚拟 DOM 节点（1~2个）：

- `<template>`
- `wx:if`
- `wx:for`

简单列举一个 Taro3 渲染与原生渲染的对比例子：

```html
<!-- 原生：2 个渲染层虚拟 DOM 节点 -->
<view>
  <view></view>
</view>
```

```html
<!-- Taro3：6 个渲染层虚拟 DOM 节点（template x2, view x2, wx:for x2） -->
<template name="tmpl_view_0">
  <view>
    <block wx:for="{{i.cn}}">
        <template is="tmpl_view_1" />
    </block>
  </view>
</template>

<template name="tmpl_view_1">
  <view></view>
</template>
```

综上，Taro3 通过模版拼接进行渲染的方式会比原生写法多 **3~4 倍**的渲染层虚拟 DOM 节点。

在小程序分层架构的限制下，Taro3 的渲染方式仍然是动态渲染节点的唯一写法，只有这样才能使真正的 React、Vue 等 Web 框架在小程序中运行起来。然而借鉴 Vue3 的思路，JSX、Vue template 中除了动态的部分外还会存在静态的部分。假设我们提前在编译阶段把静态部分识别出来，然后生成对应的小程序模板，就能使这部分对应的小程序渲染层虚拟 DOM 节点数趋近原生写法：

```jsx
// Taro 组件
function () {
  return (
    <View>
      <View></View>
    </View>
  )
}
```

```html
<!-- 优化后：3 个渲染层虚拟 DOM节点（template x1, view x2） -->
<template name="tmpl_xxx">
  <view>
    <view></view>
  </view>
</template>
```

因此，Taro 将推出**小程序编译模式（CompileMode）**，让开发者可以手动对存在性能问题的组件进行优化，例如长列表中的用于循环的 item 组件等。

## 使用案例

开发者只需要给小程序基础组件添加 `compileMode` 属性，该组件及其 children 将会被编译为单独的小程序模板：

```jsx
function GoodsItem () {
  return (
    <View compileMode>
      ...
    </View>
  )
}
```

## 详细设计

**CompileMode** 的主要工作是修改用户源码和生成对应的 `template`。这部分处理应该发生在打包工具的编译阶段，在 Babel 处理之前。Webpack Loader 和 Vite Plugin 的 `transform` 钩子都是不错的入口。而对源码的编译、解析、转换等一系列操作，我们选择了比 Babel 速度更快的 SWC。

接下来讨论对部分 JSX 语法的转换逻辑：

#### 1. 动态属性和动态子节点

编译器遇到组件的 `compileMode` 属性时，将递归处理该节点的 `attributes` 和 `children`。当属性或子节点是常量时，我们称之为静态节点，而当遇到 JSX 表达式 `{}` 时，我们称之为动态属性和动态节点。

例子：

```html
// 静态属性和静态子节点
<View hoverClass="myClass">
  <Text>Hello World!</Text>
</View>

// 动态属性和动态子节点
<View hoverClass={myHoverCls}>
  <Text>{content}</Text>
</View>
```

以下是动态部分的模板编译结果：

```html
<template name="compile-mode">
  <view hover-class="{{i.hoverClass}}">
  	<text>{{i.cn[0}.cn[0].v}}</text>
  </view>
</template>
```

可以看出，对动态部分的取值是根据其在 DOM 树的位置来决定的，节点层级越深，取值路径越长。因此大量深层级的动态部分会造成模板体积过大的问题，是编译模式后续迭代的重要课题之一。

#### 2. 对静态属性和静态子节点进行剪枝

我们在编译阶段可以提前识别出静态属性和静态子节点，然后把这类静态部分提前输出在小程序的模板上，而在源码中执行剪枝操作，这样可以避免 Taro 对静态部分进行额外的渲染和 setData，进一步提升性能。

例子：

```jsx
// 源码
<View compileMode>
  <View style="color: red" hoverStopPropagation>
      <Text>Hello World!</Text>
  </View>
  <Text>Hello{T1}World{T2}!</Text>
</View>
```

```jsx
// 剪枝后的 JSX，只保留动态部分
<View>
  <Text>{T1}{T2}</Text>
</View>
```

```xml
<!-- 模板 -->
<view>
  <view hover-stop-propagation="true" style="color: red">
    <text>Hello World!</text>
  </view>
  <text>Hello{{i.cn[0].cn[0].v}}World{{i.cn[0].cn[1].v}}!</text>
</view>
```

#### 3. 事件

事件的处理相对简单，不需要修改用户源码，只要在小程序模板上输出对应的事件绑定和 id 属性：

```jsx
// 源码
<View onClick={handleViewClick}></View>
```

```xml
<!-- 模板 -->
<view bindtap="eh" data-sid="{{i.sid}}" id="{{i.sid}}"></view>
```

#### 4. 条件表达式

JSX 中常用的条件表达式有逻辑运算符（`&&`）和三目运算符（`?:`）两种。

对于前者，需要先改造为三目运算符以提供占位元素，目的是为了在显示或隐藏时都不影响兄弟节点的动态属性路径：

```jsx
// 源码
<View compileMode>
  {condition && <View hoverClass={myClass}>{content}</View>}
</View>
```

```jsx
// 编译后的 JSX
<View>
  {condition ? <View compileIf={condition}>{content}</View> : <View/>}
</View>
```

```xml
<!-- 模板 -->
<view>
  <view wx:if="{{i.cn[0].compileIf}}">{{i.cn[0].cn[0].v}}</view>
</view>
```

对于后者，只需要添加条件控制属性 `if` 和 `else` ：

```jsx
// 源码
<View compileMode>
  {condition ? <View>{content}</View> : <Text>hello</Text>}
</View>
```

```jsx
// 编译后的 JSX
<View>
  {condition ? <View compileIf={condition}>{content}</View> : <Text compileElse>hello</Text>}
</View>
```

```xml
<!-- 模板 -->
<view>
  <view wx:if="{{i.cn[0].compileIf}}">{{i.cn[0].cn[0].v}}</view><text wx:else>hello</text>
</view>
```

#### 5. 循环表达式

开发者的代码在运行时会被执行、渲染为 Taro 虚拟 DOM 树，然后 DOM 树的数据经序列化后被 `setData` 到小程序渲染层。在渲染层的角度来看，关注的只是经序列化后的数据。因此在编译阶段，我们只需要知道哪些节点是循环节点，然后输出小程序模板的循环语法即可：

```jsx
// 源码
<View compileMode>
  {list.map(item => {
    return <View>{item}</View>
  })}
</View>
```
```jsx

// 编译后的 JSX
<View compileMode>
  {list.map(item => {
    return <View compileFor compileForKey>{item}</View>
  })}
</View>
```

```xml
<!-- 模板 -->
<view>
  <view wx:for="{{i.cn}}" wx:key="sid">{{item.cn[0].v}}</view>
</view>
```

#### 6. 兼容性

与 Taro 1、2 不同，`CompileMode` 中的 React、Vue 组件并不对应着一个个小程序自定义组件。因此在编译阶段是没有办法去处理它们的，这时应该降级到 Taro3 默认的 `template` 拼接的渲染方式。除此之外，其余不能在编译阶段处理的语法同样需要作降级渲染来进行兼容。

例子：

```jsx
// 源码
<View compileMode>
  <Counter />
</View>
```

```xml
<!-- 模板 -->
<view>
  <template is="{{'template_' + i.cn[0].nn}}" data="{{i.cn[0]}}" />
</view>
```


## 缺陷

- 小程序编译模式使用了空间来换时间，编译出模板会令包体积增大，开发者应酌情使用
- 编译阶段只能识别、优化部分语法
- 针对不同的开发框架 DSL，Taro 需要分别去做适配，如 JSX、Vue template 等，工作量会比较大

## 替代选择

N/A

## 适配策略

- Taro 将优先推出对 React JSX 的支持，后续会推出对 Vue template 的支持。
- 文档中应该注明小程序编译模式对不同语法的支持程度
- 对于在小程序编译模式中不支持的语法，自动降级为 Taro3 默认渲染模式