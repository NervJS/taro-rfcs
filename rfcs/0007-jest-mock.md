- 提案时间: 2021-08-15
- 影响版本: 3.x
- 相关 Issues: _

## 概述

为 jest 添加各种平台小程序内置 api 的 mock， 这样可以方便 taro 跑测试用例

## 动机

最近在尝试为 taro 提交 pr，发现在跑测试用例的时候，发现时间特别漫长，而且很多地方使用了 snapshot 
方式进行测试，这样虽然可以避免小白开发者的任意 pr ，但是对于增加一个新的 api 来说，还是比较繁琐的，
而且很多地方的更改是没有意义的， 

## 使用案例

首先在 `package.json` 文件中增加如下配置

```
{
  "jest": {
    ...
    "setupFiles": ["jest-mimiproject-mock"]
  }
}
```

具体的测试用例代码，可以像这样：

```
test('test showModal in weapp', ()=>{
  // 当前处于微信开发环境 ...
  Taro.showModal('12')
  expect(wx.showModal).toHaveBeenCalled()
  expect(wx.showModal).toHaveBeenCalledWith('12')
})
```

## 详细设计

这个功能可以参考 [jest-localstorage-mock](https://github.com/clarkbw/jest-localstorage-mock) 封装成一个 jest 的插件，
方便导入

## 缺陷

我们是不是可以不做这个功能，请考虑：

- 实现这个功能的投入：我可以利用业余时间来实现，不过需要有人帮忙 review 代码，防止出差错
- 跟应用层无关
- 可以加快测试用例的执行
- 不影响已发布的版本，使用者不需要考虑迁移问题
- 这属于 chore，不影响其他的 rfc 

## 替代选择

还有其他的方案也可以实现这个功能吗？
目前只想到了这个方案

## 适配策略

不需要考虑开发者适配问题