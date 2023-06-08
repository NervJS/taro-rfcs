- 提案时间: 2023-02-08
- 影响版本: 3.x

## 概述

支持 [react-server-component](https://reactjs.org/blog/2020/12/21/data-fetching-with-react-server-components.html)，以下简称 RSC

## 动机

1. 有效的减少应用体积
2. 组件内直接支持异步函数调用

## 使用案例

```jsx
// pages/index/index
import React, { Suspense, use } from "react";
import Taro from "@tarojs/taro";
import { View } from "@tarojs/components";
import { createFromReadableStream } from "react-server-dom-webpack/client";
import { TransformStream } from "web-streams-polyfill/ponyfill"; // stream polyfill
const stream = new TransformStream();

const task = Taro.request({
  url: "http://localhost:3000/react", // request RSC
  enableChunked: true,
});

task.onChunkReceived(({ data }) => {
  const writer = stream.writable.getWriter();
  writer.write(new Uint8Array(data));
});
let index = createFromReadableStream(stream.readable);

const Index = () => {
  const app = use(index);
  return <View className="wrapper">{app}</View>;
};

export default function IndexPage() {
  return (
    <Suspense fallback={<View>loading...</View>}>
      <Index />
    </Suspense>
  );
}
```

## 详细设计

### 1. 更新包

- packages
  - taro-react
    - react-reconciler@next
- 应用侧
  - react@next

### 2. 新增包

- 应用侧
  - react-server-dom-webpack
  - web-streams-polyfill

`react-server-dom-webpack/client` 提供把 stream 转换为 ReactNode 的工具。若在 taro 内部实现 RSC 则开发者不需要安装该包

### 3. webpack target 设置为 node

在 `react-server-dom-webpack` 包中由于内部使用到了 `__webpack_chunk_load__` 函数，若 webpack target 为 browser 时，其实现将会从 **document** 的 **script** 中加载脚本，而小程序中没有 **script** 属性，导致报错。

### 4. 提前渲染 ServerComponent 模板

关乎在 [实现细节](https://docs.taro.zone/docs/implement-note#%E8%BF%90%E8%A1%8C%E6%97%B6) 一章中提到的一些框架问题。Taro 需要先把 ServerComponent 渲染一遍获取模板，然后填入 `base.xml` 中，这意味着我们无法省下模板的体积，但依然可以省下组件中引入 js 库的那一部分体积。

## 缺陷

1. 拥有 RSC 所有的缺点
   - RSC 无法交互
   - RSC 无法拥有状态
   - 开发者需要服务器，并运行支持 RSC 渲染的服务
   - ... 更多可以查看[这里](https://beta.nextjs.org/docs/rendering/server-and-client-components)
2. 由于小程序原生并不支持 Stream API，所以需要引入 polyfill

## 替代选择

N/A

## 适配策略

在 Taro 内部实现 RSC，开发者可以无感知的使用 RSC 的特性
