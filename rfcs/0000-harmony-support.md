- 提案时间: 2023-11-24
- 影响版本: 3.x
- 相关 Issues: N/A

## 概述

在 Taro 中实现对 HarmonyOS 中 ArkTS 语言和 ArkUI 的支持，开发者可以通过 Taro CLI 构建 HarmonyOS Stage 模型的项目代码。

## 动机

实现 HarmonyOS 中对 ArkTS 语言的支持，来吸引更多喜欢尝鲜追新的开发者使用 Taro。

HarmonyOS 在 [2022 年 9 月](https://developer.harmonyos.com/cn/docs/documentation/doc-releases-V3/harmonyos_release_3_202209-0000001323985308-V3) 发布 3.0 正式版，在 3.0 正式版中，全新发布基于 TS 扩展的声明式开发范式，支持 TypeScript 编程语言，采用链式调用，提供装饰器和 MVVM 能力，支持条件、循环渲染、懒加载渲染，支持自定义组件，旧的 JSUI 模式将不再提供维护，意味着 JSUI 将逐渐退出历史舞台，ArkUI 将成为未来鸿蒙系统开发应用的主流选择。

## 使用案例

目前在 Taro 项目中只支持使用 React 来进行 Harmony 项目的开发，在页面组件和自定义组件开发中，在 Taro 中写 Harmony 项目就和写 React 项目遵循[一样的规范](https://nervjs.github.io/taro-docs/docs/react-overall)，但具体的语法是否支持取决于当前对于 ArkUI 相关组件和 API 的支持程度。

```ts
import { PropsWithChildren } from 'react'
import './app.scss'

function App({ children }: PropsWithChildren<any>) {
  // children 是将要会渲染的页面
  return children
}

export default App
```

## 详细设计

![原理图](https://img14.360buyimg.com/img/jfs/t1/103172/22/36133/166586/65606352Fd96bff74/aacc6219adad4dcb.png)

### 1. 在鸿蒙环境下模拟一套浏览器的 BOM 和 DOM
需要支持使用 React 的语法来开发鸿蒙应用，首先我们需要模拟一套浏览器的 BOM 和 DOM，比如一些 windows、history、document 等全局对象，来给到 React 框架层去使用，使的这些依赖浏览器环境的 Taro 代码可以跑在鸿蒙 app 上。
![DOM](https://img13.360buyimg.com/imagetools/jfs/t1/230031/37/4320/279939/6560682eF9b22521a/5efd882e262dd66d.png)

### 2. 通过 React-Reconciler 接入这些 BOM、DOM
之后我们和小程序类似的，在 React-Reconciler 里，通过定义一个自定义的 hostconfig，[将 React 一些关键的增删改查宿主环境节点的操作与我们构建的虚拟 BOM、DOM 相绑定。](https://github.com/NervJS/taro/blob/next/packages/taro-react/src/reconciler.ts)绑定成功后，React 项目代码的结果就会直接反应到我们创建的 Taro 虚拟 BOM、DOM 中了。

### 3. 实现虚拟 BOM、DOM 到 ArkTS 的桥阶层
目前为止，在执行完 React 框架层代码后，我们便能得到一颗由这些虚拟的 DOM 组成的一颗虚拟 DOM 树。

之后我们会将这颗 DOM 树传递给页面入口，绑定在其成员属性 node 上，触发渲染，然后 ArkTS 就会根据这颗 node 树进行一次递归的渲染调用，生成对应的原生组件，从而渲染出具体的页面。

![原理图](https://img14.360buyimg.com/imagetools/jfs/t1/236593/13/4023/141407/6560835eFead86b40/c19b31544ab9d37c.png)

在用户在触发一些交互行为的时候，会导致上层的 React 框架调用一些修改 DOM 树节点和 DOM node 属性的逻辑，那么这些 DOM 由于是 Taro 自身模拟的，[在创建时都会绑定上 Observed 装饰器，这个装饰器会配合 @objectLink 装饰器来使用](https://developer.harmonyos.com/cn/docs/documentation/doc-guides-V3/arkts-observed-and-objectlink-0000001473697338-V3)，去监听 Taro DOM Node 上属性的变化，并触发所在组件更新方法的调用，从而达到更新的目的。

```ts
// TaroVideoElement
@Observed
class TaroVideoElement extends TaroElement {
  constructor() {
    super('Video')
  }

  async play() {
    try {
      this._instance.controller.start()
      return Promise.resolve()
    } catch (e) {
      return Promise.reject(e)
    }
  }
  // ...
}

// TaroVideoArkComponent
@Component
export default struct TaroVideoArkComponent {
  @ObjectLink node: TaroVideoElement
  build() {
    Video(this.getVideoData(this.node))
      .defaultEvent()
      .props(this.getVideoProps())
      .attrs(getNormalAttributes(this.node))
  }
}

```

### 4. 使用 ArkTS 实现 Taro 组件和 API 标准里包含的内容
从上面的流程可以看出，每种类型的 React HostComponent 组件，到最后都需要对应一种类型的 TaroElement 以及一种类型的 ArkComponent，因此我们需要针对这些不同类型的 TaroElement 和 ArkComponent 都需要有一套完整的实现。

除了组件外，API 也需要遵循 Taro 目前的标准，利用鸿蒙环境提供的 API，去模拟和实现一套 Taro 标准的 API，并将这些 API 绑定在 Taro 的全局对象上导出供用户使用。

### 5. 实现鸿蒙平台的工程化逻辑
在实现了运行时适配代码后，还需要处理工程化相关内容，这里主要分为以下三块：
#### (1) 项目编译打包逻辑
我们需要利用打包工具，将用户所写的 JSX 通过 babel 等工具转译成 Harmony ArkTS 可以读懂的 TypeScript/JavaScript 语言。

并将 Taro 适配鸿蒙环境的胶水代码、组件和 API 等运行时代码拷贝到鸿蒙的工程目录下，最后在鸿蒙的工程目录下，根据用户配置，完成对页面入口文件的初始化。
#### (2) 样式解析逻辑
其次，ArkTS 不支持 CSS 文件，因此我们还需要利用打包工具对样式文件进行处理，我们会在编译时分析出所有引用了 CSS 文件的 JSX 和 TSX 代码。

然后我们会[利用 Rust 开发一个解析 React 组件与对应的 CSS 文件的工具](https://github.com/NervJS/parse-css-to-stylesheet)，为每一个 React 节点计算样式最终样式，应用于 React Native、鸿蒙等不支持 CSS 写法的场景（目前仅支持类名选择器）。

最后我们会将这些引用了 CSS 文件的代码交给这个样式解析工具，工具会将这些样式以 Style 属性的方式写在这些 JSX 节点上面，返回最终处理后的 JSX 和 TSX 代码给后续的编译操作。 
#### (3) 半编译逻辑
由于我们的方案在运行时环节会初始化很多的自定义组件实例，因此我们这个方案的主要耗时都消耗在了这个实例化逻辑上面，因此我们在编译时会通过类似[小程序半编译方案](https://github.com/NervJS/taro-rfcs/blob/feat/compile-mode/rfcs/0000-compile-mode.md)的方式，将一些可以提前分析的代码节点生成对应的模板文件，从而减少最后页面渲染时实例化自定义组件的数量。

## 缺陷

1. 样式解析存在限制，只支持类名选择器，并且不支持样式的继承和级联，且引入的样式只能作用于当前的 JSX 和 TSX 文件。
2. 组件和 API 都需要重新实现，因此部分 Taro 标准里的功能在鸿蒙环境下可能没办法支持，以及后续标准如果出现更新，组件和 API 也需要同步进行更新。
3. 目前的方案还未对混合编译模式等在原本 Taro 项目中较为常见的编译模式进行支持。
4. 暂时不支持动态化。

## 替代选择

N/A

## 适配策略

1. 完善文档；
2. 新增 HarmonyOS ArkTS 模板；
3. 发布推广文章和框架原理介绍文章；
