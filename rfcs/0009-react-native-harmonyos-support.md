- 提案时间: 2023-05-18 
- 影响版本: 3.x
- 相关 Issues: N/A

## 概述

通过RN鸿蒙化，以及实现Taro-RN所依赖的RN三方库，从而完成Taro的鸿蒙化。

## 动机

使用Taro-RN的应用如果需要鸿蒙化，并且为了尽可能减少应用代码的修改量，那么比较好的做法是将RN鸿蒙化，然后通过[Taro RN 支持](./0003-react-native-support.md)可以完成Taro-RN的鸿蒙化，最终达到Taro-RN应用快速鸿蒙化的效果。

## 使用案例

如果提案新增或更改 API ，需要带上一个基本的代码案例。
Taro-RN鸿蒙化的使用案例遵循[现有的模式](https://docs.taro.zone/docs/react-native#%E9%9B%86%E6%88%90%E6%A8%A1%E5%BC%8F)，其中打包js bundle

```
yarn build:rn --platform openharmony
```

对应的启动命令

```
yarn openharmony
```

## 详细设计

设计思路按照[React Native 端开发流程](https://docs.taro.zone/docs/react-native)展开

1. 在[模版工程](https://gitee.com/o2team/taro-project-templates/tree/v3.6/react-native)中新增OpenHarmony的实现，复用RN鸿蒙化的工作。
2. RN metro新增OpenHarmony打包，复用RN鸿蒙化的工作。
3. 样式转换不需要新增功能。
4. 由于RN鸿蒙化的版本是0.71.3，并且采用的RN新架构，意味着Taro-RN使用的三方库鸿蒙化适配需要使用新架构。

新增RN三方库的鸿蒙化的列表
| RN三方库 | 优先级 | 工作量 |
| --- | --- | --- |
| react-native-gesture-handler | 高 | 高 |
| react-native-safe-area-context | 高 | 低 |
| react-native-screens | 中 | 中 |
| react-native-pager-view | 中 | 高 |
| react-native-reanimated | 低 | 高 |
| react-native-svg | 中 | 高 |
| react-native-image-pan-zoom | 中 | 极低，纯JS |
| react-native-image-resizer | 中 | 低 |
| react-native-zoom-viewer | 中 | 极低，纯JS |
| react-native-known-styling-properties | 中 | 极低，纯JS |
| react-native-mocker | 低 | 低，纯JS |
| react-native-modal-popover | 中 | 低，纯JS |
| react-native-root-modal | 中 | 低，纯JS |
| react-native-root-sibling | 中 | 低，纯JS |
| react-native-root-toast | 中 | 低，纯JS |
| react-native-segmented-control | 中 | 中 |
| react-native-stylekit | 中 | 低，纯JS |
| react-native-svg-transformer | 中 | 低，纯JS |
| react-native-syan-image-picker | 中 | 中 |
| react-native-test-app | 低 | 低，纯JS |
| react-native-a11y | 低 | 低，纯JS |
| react-native-builder-bob | 中 | 低，纯JS |
| react-native-cameraroll | 低 | 高 |
| react-native-collapsible | 低 | 低，纯JS |
| react-native-device-info | 高 | 低 |


| Expo三方库 | 优先级 | 工作量 |
| --- | --- | --- |
| expo | 待评估 | 待评估 |
| expo-application | 待评估 | 待评估 |
| expo-asset | 待评估 | 待评估 |
| expo-av | 待评估 | 待评估 |
| expo-barcode-scanner | 待评估 | 待评估 |
| expo-brightness | 待评估 | 待评估 |
| expo-camera | 待评估 | 待评估 |
| expo-cli | 待评估 | 待评估 |
| expo-constants | 待评估 | 待评估 |
| expo-error-recovery | 待评估 | 待评估 |
| expo-file-system | 待评估 | 待评估 |
| expo-font | 待评估 | 待评估 |
| expo-image-loader | 待评估 | 待评估 |
| expo-image-picker | 待评估 | 待评估 |
| expo-internal | 待评估 | 待评估 |
| expo-keep-awake | 待评估 | 待评估 |
| expo-location | 待评估 | 待评估 |
| expo-sensors | 待评估 | 待评估 |
| expo-web | 待评估 | 待评估 |



## 缺陷

1. 由于短期RN鸿蒙化适配不能合入到RN社区，下载RN代码不能通过npm来拉取
2. 由于RN鸿蒙化仅支持新架构，因此新增RN三方库都需要采用新架构

## 替代选择

N/A

## 适配策略

1. 已有Taro-RN的应用进行鸿蒙化，只需要将平台依赖的部分进行鸿蒙化即可完成适配。
2. 还未使用Taro-RN开发的应用适配策略可以参考[Taro RN 支持](./0003-react-native-support.md)的适配策略

