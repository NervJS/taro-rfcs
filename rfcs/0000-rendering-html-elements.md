- 提案时间: 2020-06-26
- 影响版本: 3.x
- 相关 Issues: N/A

## 概述

在 Taro 中实现直接渲染 HTML 标签的功能。

## 动机

允许开发者直接使用 HTML 标签，进一步抹平小程序与 H5 之间的差异，使社区有机会尝试整合一些现存的 Web UI 库。

## 使用案例

```jsx
import NoCode from 'react-no-code'

export default () => (
    <div>
        <main>
            <button>Getting Start</button>
            <NoCode />
        </main>
        <footer>Copyright</footer>
    </div>
)
```

## 详细设计

React 天然支持 HTML 标签，无需导入，类型齐全。

由于之前已经实现了 `dangerouslySetInnerHTML` 渲染 HTML 的功能和样式，只需要借用这些现有的组件，与 HTML 标签之间做一下映射。或者直接与 `@tarojs/components` 中的组件进行映射。最好是能一对一的替换，配合少量 css，例如：

| HTML                       | 小程序    |
| -------------------------- | --------- |
| div / section / p / h1 ... | View      |
| span / em / i / strong ... | Text      |
| button                     | Button    |
| a                          | Navigator |
| img                        | Image     |

HTML 标签应该尽量在功能和样式上与浏览器对齐，小程序特有的能力无需考虑，小程序缺失的能力看情况弥补，保持方案简单，不在 HTML 标签上增加额外 API，不能支持的标签降级为 View，100% 兼容 Web 标准是不现实的。

用户可以对映射关系进行配置，覆盖官方实现，作为对复杂标签的一种补充，例如 `<area>`。

## 缺陷

- 一些 HTML 标签因样式差异较大难以适配，例如 `<button>` `<video>`
- 会导致滥用，用户不想再导入 View 和 Text 了，而是倾向于 `<div>` 和 `<span>`。

## 替代选择

用户自行 Monkey patch `React.createElement`。

## 适配策略

考虑使用插件实现，把选择权交给用户。
