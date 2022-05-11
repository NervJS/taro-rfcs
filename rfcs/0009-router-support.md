- 提案时间: 2022-5-10
- 影响版本: 3.x

## 概述
Taro 目前已经支持使用 React/Vue 前端框架开发小程序，Taro 为了支持这些前端框架的使用，维护了一个运行时，包含了常见 BOM/DOM 的 API 。但目前 Taro 运行时暂时还没有对 `history/location` 进行支持，导致目前常见的前端路由解决方案暂时无法使用，如 `react-router`、`vue-router` 等。             
本提案旨在提供一个解决方案以支持路由功能。

## 动机
Taro 增加 `history/location` 的运行时实现，以支持常见路由方案能够在 Taro 中落地实施，提升开发体验和完善功能。

## 使用案例

### 前提
因路由库内部使用了 html 标签，因此需要使用 [`@tarojs/plugin-html`](https://docs.taro.zone/docs/use-h5) 插件支持在 Taro 中使用 H5 标签开发小程序。         

本提案期望能支持如下常见两种路由方案：

### react-router
```js
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom'
export default class Index extends Component {
  render() {
    return (
      <BrowserRouter>
        <View className='drawer-box'>
          <View className='box-item'>
            <Link to="/pages/router/index/view1?a=1&b=2">
              view1
            </Link>
          </View>
          <View className='box-item'>
            <Link to="/pages/router/index/view2#a=3&b=4">
              view2
            </Link>
          </View>
          <View className='box-item'>
            <Link to="/pages/router/index/2?a=1&b=2#a=3&b=4">
              view3
            </Link>
          </View>
        </View>

        <Routes>
          <Route path="/pages/browser-router/index" element={<Home/>}></Route>
          <Route path="/pages/router/index/view1" element={<View1/>}></Route>
          <Route path="/pages/router/index/view2" element={<View2/>}></Route>
          <Route path="/pages/router/index/:id" element={<View3/>}></Route>
        </Routes>
      </BrowserRouter>
    )
  }
}
```

### vue-router
app.js      
```js
import { createApp } from 'vue'
import { createWebHistory } from 'vue-router'
// 自定义组件
import Tab1 from './components/tab-1.vue'
import Tab2 from './components/tab-2.vue'
import Tab3 from './components/tab-3.vue'
import User from './components/user.vue'
import Home from './components/home.vue'

const routes = [
  { path: '/', component: Home },
  { path: '/tab1', component: Tab1 },
  { path: '/tab2', component: Tab2 },
  { path: '/tab3/:id', component: Tab3 },
  { path: '/user/:username', name: 'user', component: User}
]

const router = createRouter({
  history: createWebHistory(),
  routes
})

const App = createApp({
  onShow (options) {},
})

App.use(router)

export default App
```
在具体页面中使用路由：
```html
<template>
  <view>
    <view>
      <a href="../index/index">Go to Index</a>
      <view @click="showHistory">show history</view>
      <view @click="showLocation">show location</view>
      <view class="tab-box">
        <router-link class="tab-item" to="/" replace><text>Home</text></router-link> |
        <router-link class="tab-item" to="/tab1?name=lucy" replace>Tab 1</router-link> |
        <router-link class="tab-item" to="/tab2">Tab 2</router-link> |
        <router-link class="tab-item" to="/tab3/1234">Tab 3</router-link> |
        <router-link class="tab-item" :to="{ name: 'user', params: { username: 'lucy' }}">User</router-link> |
      </view>

      <router-view></router-view>
    </view>

  </view>
</template>

<script setup>
  function showHistory(){
    console.log('history', window.history)  // 能够打印出正确的history对象
  }
  function showLocation(){
    console.log('location', window.location)  // 能够打印出正确的location对象
  }
</script>

```

## 详细设计

### 1、 在`@tarojs/runtime`中新增 location 对象的实现
```js
class Location extends Events {
    get host(){}
    set host(){}
    // more properties...

    replace(){}
    // more methods...
}
```
Location 继承 Events，以实现事件观察

### 2、 在`@tarojs/runtime`中新增 history 对象的实现
```js
class History extends Events {
    get state(){}
    // more properties...

    pushState(){}
    // more methods...
}
```
History 继承 Events，以实现事件观察


### 3、 window 改造   
#### 3.1、 抽象 Window 类
主要目的是实现事件观察机制，便于逻辑封装。
```js
class Window extends Events {
    constructor(){
        // 向window对象中挂载属性
        this.document = document
        this.document.defaultView = this
        this.navigator = navigator
        this.requestAnimationFrame = raf
        this.cancelAnimationFrame = caf
        this.getComputedStyle = getComputedStyle
    }
    addEventListener(event, cb){
        this.on(event, cb)
    }
    removeEventListener(){

    }
}
// 在Taro运行时仍保持单例
export const window = new Window()
```

#### 3.2、 对象上新增 `addEventListener` `removeEventListener` 方法
`window.addEventListener` 用于监听 `hashchange`/`popstate` 事件，并触发回调使用。                 
```js
class Window extends Events {
    addEventListener(event, cb){
        this.on(event, cb)
    }
    removeEventListener(){
        this.off(event, cb)
    }
}
```

#### 3.3、 window对象上挂载 `location&history` 对象
```js
window.location = new Location()
window.history = new History()
```

### 4、页面历史信息上下文切换
小程序是多页面架构，当页面跳转时，小程序新开或当前打开新页面，此时需要对页面的历史信息进行处理。借助于页面的生命周期，可以做到历史信息的上下文切换：

```js
// packages/taro-runtime/src/dsl/common.ts
const config = {
    onLoad(){
        // 初始化当前页面的上下文信息
        window.trigger(CONTEXT_ACTIONS.INIT, this.$taroPath)
    }
    onUnload(){
        // 销毁当前页面的上下文信息
        window.trigger(CONTEXT_ACTIONS.DESTORY, this.$taroPath)
    }
    onShow(){
        // 恢复上下文信息
        window.trigger(CONTEXT_ACTIONS.RECOVER, this.$taroPath)
    }
    onHide(){
        // 缓存当前页面上下文信息
        window.trigger(CONTEXT_ACTIONS.RESTORE, this.$taroPath)
    }
}
```

## 缺陷
1. 在Web端可以通过赋值 `location.href` 实现页面加载，但在小程序中不适用，小程序端的页面跳转仍建议使用 `navigateTo` 等官方 api。在小程序侧，应该将location上的属性视为只读。
2. location.origin 将默认为 `https://taro.com` ; `location.assigin()` `location.replace()` 在小程序侧无效；
3. 由于 `a` 标签在Taro中会被翻译为 `navigator` 组件，因此开发者在设计路由时，因尽量避免使用页面同名，否则会触发页面跳转；
4. 需要更多案例测试；

## 替代选择

N/A

## 适配策略
- 提供详尽的使用教程与配置文档。