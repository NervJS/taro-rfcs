- 提案时间: 2023-06-01
- 影响版本: (3.x)

## 概述

支持开发者开发 Taro 应用时，可以进行组件/应用级别的单元测试，提高代码健壮性。

## 动机

### 统一跨框架的测试工具 API

目前针对组件级别的测试，在使用`react`框架时，可以采用`react-test-renderer`,`vue`框架可以采用`@vue/test-utils`这类官方提供的单元测试工具。
我们可以针对这种情况，在各框架提供的官方测试工具能力基础上进行封装，可以使用一套 api，进行跨框架层面的测试

### 覆盖应用级的测试

为什么：Taro 的入口组件、页面组件都有自身的生命周期。单纯使用 react/vue 框架的 test-utils 无法实现 Taro 生命周期的执行流程。
场景：需要进行路由切换测试、入口/页面组件生命周期测试，小程序生命周期等
结果：实现在测试环境下从入口文件开始，模拟完整的 Taro 应用页面渲染结果

## 使用案例

### 组件级别

```javascript
// __test__/main/index.test.js
import TestUtils from "@o2team/test-utils-react";
import Hello from "../../src/components/Hello.tsx";
const testUtils = new TestUtils();
describe("App", () => {
  it("RenderComponent", async () => {
    // React跟Vue相同用法
    await testUtils.mount(Hello, {
      // 配置项
      props: {
        a: 1,
      },
    });
    // 等待页面出现.btn这个节点
    const btn = await testUtils.queries.waitForQuerySelector(".btn");
    // 等待react的渲染更新完成
    await testUtils.act(() => {
      // 触发点击事件
      testUtils.fireEvent.click(btn);
    });
    // 打印渲染结果
    expect(testUtils.html()).toMatchSnapshot();
    // <div class="hello">...
  });
});
```

### 应用级别

```javascript
// __test__/main/index.test.js
import TestUtils from "@o2team/test-utils-react";
import App from "../../src/app.ts";
const testUtils = new TestUtils();
describe("App", () => {
  it("RenderApp", async () => {
    await testUtils.createApp(App, {
      // 配置项
      // pages: [
      //   'pages/index/index',
      //   'pages/second/index',
      // ]
    });
    // 监听/pages/index/index这个页面路由的onShow生命周期触发
    await testUtils.PageLifecycle.onShow("/pages/index/index");
    // 跳转到第二个页面
    Taro.navigateTo({ url: "/pages/second/index" });
    // 监听/pages/second/index这个页面路由的onShow生命周期触发
    await testUtils.PageLifecycle.onShow("/pages/second/index");
    // 当/pages/second/index触发onShow后，打印页面内容
    expect(testUtils.html()).toMatchSnapshot();
    // <body><div class="taro_router" id="app">...
  });
});
```

## 详细设计

### 1. 框架层、DOM 层的分离设计

参考 Testing Library 的设计思路，将框架层和 dom 层分离；

#### 1.1 DOM 层负责：节点查询、事件触发的机制

- 提供对于 document 的查询 api，比如选择器查询、文本查询，waitFor 监听节点出现等
- 提供事件触发的 api，通过 dispatch 事件，触发框架层节点的事件监听响应

#### 1.2 框架层负责：组件的挂载和渲染

- 视情况各框架对 dom 层提供的 api 进行自身改造重写
- 暴露 mount 函数挂载组件并渲染到指定的 dom 上，react 可通过 react-dom 来渲染，vue 通过@vue/test-utils 在**jsdom**环境下上渲染
- 提供对应的初始化配置，比如 jest 的配置、运行时配置

### 2. 生命周期的 hook

劫持 AppInstance、PageInstance 所提供的生命周期函数，比如 onLoad,onShow 等，往里面加塞监听钩子，让开发者能在 jest 下监听任何生命周期的回调

### 3. web\小程序的区分

模拟小程序的渲染及生命周期，采用 h5 的渲染流程来进行测试，主要区别在于

#### 3.1 process.env.TARO_ENV

通过 jest 的 tranformer，在编译阶段处理。因为采用以 web 的形式 mock 小程序流程，所以关于@tarojs 的 node_modules 保持不变，继续走 h5 逻辑，其余都会替换为对应的环境

```javascript
// 劫持替换 @tarojs/taro => tarojs/taro-h5 + react runtime
import apiLoader from "@tarojs/plugin-framework-react/dist/api-loader.js";
if (sourcePath.includes("匹配taro-h5")) {
  sourceText = apiLoader(sourceText);
}
```

#### 3.2 组件差异

小程序还是跟 h5 一样走 stencil 的 webcomponent 的生命周期和事件响应，但是在输出渲染结果时，按照将`<taro-view-core>`输出为小程序标签`<view>`

### 4、运行时预处理

#### 4.1 @tarojs/taro

通过 jest 的 transformer，将`@tarojs/taro`指向到`@tarojs/plugin-framework-框架/dist/api-loader.js` + `@tarojs/taro-h5`

#### 4.2 @tarojs/component

将`'@tarojs/components$`映射为`@tarojs/components/lib/框架/index`

## 缺陷

- 缺少一个真实的小程序环境运行，所以跑的实际是 h5 的逻辑，只不过在标签的输出，环境变量，生命周期等尽可能保持逻辑流程层面一致，部分小程序特有生命周期通过测试工具暴露的方法，手动进行触发。
- 没有编译时的参与，存在某些能力在真实情况需要搭配编译插件才能实现的可能性。
- 暂未考虑 taro rn

## 替代选择

应用级测试，走 e2e 的方式，编译完后 h5 跑在 puppeteer，小程序跑在开发者工具上进行测试。

## 适配策略

- 引入`@tarojs/test-utils-xxx`测试工具，使用 jest 测试。
- 在工程上添加`jest.config.js`配置文件
