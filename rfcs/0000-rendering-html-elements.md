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

由于之前已经实现了 `dangerouslySetInnerHTML` 渲染 HTML 的功能和样式，只需要借用这些现有的组件，与 HTML 标签之前做一下映射。

React 天然支持 HTML 标签，无需导入，类型齐全。

## 缺陷

- 一些 HTML 标签因样式和功能差异较大难以适配，例如 `<button>` `<video>`
- 会导致滥用，用户不想再导入 View 和 Text 了，而是倾向于 `<div>` 和 `<span>`。

## 替代选择

用户自行 Monkey patch `React.createElement`。

## 适配策略

或许可以在配置中增加一个开关项，建议不加，如果有请默认开启。
