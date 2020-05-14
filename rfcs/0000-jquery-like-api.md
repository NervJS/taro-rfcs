- 提案时间: 2020-05-13
- 影响版本: 3.x
- 相关 Issues: 无

## 概述

可以在 Taro 中使用类似 jQuery 的 API：以 `$()` 函数为核心，以及其它众多可以链式调用的功能函数为辅助，通过直接修改 DOM 实现视图更新。

## 动机

提供 jQuery-like 的 API 并不是历史的倒退，相反，有许多场景通过 jQuery-like 的 API 可以很快地完成：

- 使用 jQuery API 构建的插件或组件库，可以在小程序和 H5 平台跨框架调用。Taro 社区开发者可以基于 jQuery API 完善 Taro 生态，同时也不会割裂 React 和 Vue 用户群
- jQuery API 并不像 React 和 Vue 这样非此即彼的选择。在特定的情况下，jQuery API 可以应用在一个 React 组件或 Vue 组件中，作为 React/Vue 开发的一种补充。（但多数情况下，使用 React/Vue 都不推荐直接操纵 DOM）
- Taro 的用户群体有一部分开发者并不十分了解现代前端框架，以及用数据驱动视图的理念
- jQuery-like API 是久经考验、广为人知的 API，用户学习成本低，一些现代前端工具（例如 puppeteer, cheerio）也都选择了 jQuery-like API

## 使用案例

### 构建一个可以跨框架跨端调用的插件/组件

可以使用 jQuery API 实现一个 `Modal` 组件：

```js
import { Current } from '@tarojs/taro'
import './modal.css'

class Modal {
  constructor ({ html, title }) {
    this.title = title
    this.modal = $(`
      <view class="popup-modal shadow" >
        <icon class="fas fa-2x fa-times text-white bg-primary p-3 popup-modal__close"></icon>
        <text class="font-weight-bold">
          ${title}
        </text>
        <view class="modal-content">
        </view>
      </view>
    `)

    this.content = this.modal.find('.modal-content')

    this._insertToCurrentPage()
  }

  _insertToCurrentPage () {
    const $page = $(Current.page.path)
    $page.append(this.modal)
  }

  show () {
    this.modal.show()
  }

  hide () {
    this.modal.hide()
  }

  changeContent (html) {
    this.content.html(html)
  }
}

export default Modal
```

这个 `Modal` 组件可以在任意一个框架中使用：

```jsx
import Modal from 'my-npm-modal'

class Home extends React.Component {
  modal = new Modal({
    title: 'Test modal',
    html: 'hello world!'
  })

  render () {
    return (
      <>
        <Button onClick={() => this.modal.show()}>Open Modal</Button>
        <Button onClick={() => this.modal.hide()}>Close Modal</Button>
        <Button onClick={() => this.modal.changeContent("<image src='https://...'>")}>Change Modal</Button>
      </>
    )
  }
}
```

### 作为现代前端开发框架的补充

```vue
<template>
  <view class="container">
    <input id="test" type="number" />
    ...
  </view>
</template>

<script>
import { $ } from '@tarojs/extend'
export default {
  mounted () {
    // 将最接近 input#test 的 text.detail 元素颜色设置为红色
    const $input = $('#test')
    $input.closest('text.detail').css({ color: 'red' })
  }
}
</script>
```


## 详细设计

实现该方案不需要编译时配合，纯运行时实现即可。核心 API 放在新的包：`@tarojs/extend`，作为 ES6 Module 导出方便 tree-shaking。`@tarojs/extend` 以后可能会承载其它额外功能。

整体来说需要在 `@tarojs/runtime` 实现三个 API：

* `document.getElementsByClassName`
* `document.getElementsByTagName`
* `$.insertToPage(content: collection)`：安全地把一个 jQuery `collection` 实例插入到当前渲染页面，类似于 `React.Portal`

在 `@tarojs/extend` 实现类似于 [zepto](https://zeptojs.com) 的功能集，但比起 `zepto` 而言会删减部分在 ES6 中已经有替代或在小程序端不可用的实现。

在具体的代码实现中，如果在不太影响 `@tarojs/runtime` 的性能和打包体积的情况下，可以考虑在 `@tarojs/runtime` 写兼容代码，直接让开发者使用 `zepto`。


## 缺陷

1. 对现代前端框架不了解的开发者，可能会滥用 jQuery API 操纵 DOM，导致代码可维护性或性能降低；
2. 作为一种构建跨框架组件/插件的方案，开发效率不高，使用起来也不够直观，未来还需要探索其它开发效率更高，更直观的跨框架组件方案

## 替代选择

1. 在 `@tarojs/runtime` 中实现完整的 DOM 支持，让开发者使用 Vanila JavaScript 的方案实现同样的功能。由于许多熟悉现代前端框架的开发者并不需要类似手段操纵 DOM，实现这套支持就相当于增加了他们应用的打包体积，同时也可能会降低现有 Runtime 的性能。Vanila JavaScript 的 API 设计也不如 jQuery 的 API 设计深入人心。
2. 直接引入原生 jQuery。原生 jQuery 有太多浏览器兼容代码（多数是 IE 的），这在小程序当中是完全不必要的。另外引入原生 jQuery 同样需要大量拓展 `@tarojs/rutmime`，和带来和选择 1 一样的问题。另外，原生 jQuery API 过于繁杂，保证和测试每个 API 都可用工作量过大，如果有不可用的 API 会给开发者带来困扰。


## 适配策略

作为现代前端框架的补充 API，需要提醒开发者使用数据去驱动视图更新是更好的解决方案。

作为实现跨框架组件/插件的解决方案，帮助社区适配该方案还需要一些额外的工作：

1. 在文档中把所有 API 功能/参数都详细列出来；
2. 单独开一章文档讲解如何用 jQuery like API 实现跨框架组件/插件；
3. 使用 jQuery like API 实现一个真正可用的组件或插件；
4. 在 3.0 版本发布和后续的文章介绍/推广中有意识提及该方案的存在和用法
