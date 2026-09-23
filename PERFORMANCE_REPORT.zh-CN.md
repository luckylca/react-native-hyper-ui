# Hyper UI 性能优化报告

**报告日期：** 2026-09-23
**范围：** `react-native-hyper-ui` 组件库的 React 渲染、列表子项更新和手势对象创建。
**目标：** 减少重复的 JavaScript / React 工作，同时保持组件 API、布局、颜色、动画曲线和触摸行为。

## 结论摘要

本轮优化分两部分完成。

1. 第一部分为常用基础组件增加浅比较 memo，并针对 Slider 的密集刻度点降低重复渲染。一个隔离的 React 协调器基准中，120 个稳定 Text 行在 30 次父组件更新期间，Text 函数调用减少 **96.8%**、原生文本节点属性提交减少 **99.2%**、协调时间中位数减少 **54.8%**。归一化后的宿主树 SHA-256 完全一致。
2. 第二部分继续给剩余常用控件增加 memo，拆分导航项和分段选项，并让 Slider 在受控值变化时复用同一个 Pan 手势对象。该部分通过包与应用类型检查、组件库构建；**没有采集本轮改动后的真机 FPS 或帧时间数据**，因此不把这些改动表述为已验证的 FPS 提升。

动画 worklet、弹簧与 timing 参数、手势阈值、颜色、尺寸和视觉 tokens 没有调整。设备截图检查显示第一部分优化前后的 onboarding 内容几乎一致；该截图检查没有覆盖组件展示页或滚动性能。

## 优化范围与实现

### 第一部分：避免稳定组件随父组件重复渲染

以下组件使用 `React.memo` / `memo`：

- 基础显示：`Text`、`Icon`、`Surface`、`Divider`
- 常用组合控件：`ListRow`、`Button`、`Card`、`Switch`、`Slider`

当 props 浅比较相同时，React 可以跳过该组件函数执行和子树协调。props 发生变化时仍正常更新；组件读取的主题或减少动效 context 改变时仍会正常更新。组件内部状态更新也不会被 memo 阻断。

### 第一部分：减少 Slider 刻度点重复工作

- 使用稳定的空刻度点常量，避免未传 `keyPoints` 时每次渲染产生新的默认数组。
- 对显示刻度点列表使用 `useMemo`，仅在刻度配置改变时重新生成。
- 将刻度 marker 拆成 memo 子组件。Slider 数值改变时，位置和激活状态未变的刻度点可以跳过 React 渲染。

涉及代码：[`Slider.tsx`](src/components/Slider.tsx)。

### 第二部分：补齐其余控件的 memo 边界

为 `Checkbox`、`Input`、`ProgressIndicator`、`NumberWheel`、`NavigationBar`、`SegmentedControl`、`TopAppBar`、`SearchBar`、`Dialog`、`BottomSheet`、`Menu` 和 `Snackbar` 增加浅比较 memo。组件内部状态和 context 仍按原有方式工作；当父组件更新但这些组件的 props 没变时，React 可以跳过该控件的重复渲染。

涉及代码：[`Checkbox.tsx`](src/components/Checkbox.tsx)、[`Input.tsx`](src/components/Input.tsx)、[`Dialog.tsx`](src/components/Dialog.tsx)、[`BottomSheet.tsx`](src/components/BottomSheet.tsx)、[`Menu.tsx`](src/components/Menu.tsx)、[`NavigationBar.tsx`](src/components/NavigationBar.tsx)、[`SegmentedControl.tsx`](src/components/SegmentedControl.tsx)、[`TopAppBar.tsx`](src/components/TopAppBar.tsx)、[`SearchBar.tsx`](src/components/SearchBar.tsx)、[`Snackbar.tsx`](src/components/Snackbar.tsx)、[`NumberWheel.tsx`](src/components/NumberWheel.tsx) 和 [`ProgressIndicator.tsx`](src/components/ProgressIndicator.tsx)。

### 第二部分：缩小选项变化时的渲染范围

导航栏项目和分段选择项现在分别是 memo 子组件。输入数组、回调和单项数据引用稳定时，选中项变化只改变旧选中项和新选中项的 `selected` 属性；其余项目可以跳过渲染。窗口尺寸、主题或传入引用改变时，相关组件仍会更新。

此效果依赖调用方传入稳定的数组、对象和回调。如果页面每次 render 都重新创建项目数组、对象或回调，浅比较会识别为变化，memo 的收益会减少。

### 第二部分：复用 Slider 手势对象

Slider 的 Pan 手势和工作回调现在按交互配置使用 `useMemo` / `useCallback`。只更新受控 `value` 时，手势对象可以保持稳定；禁用状态、范围、step、刻度点、阈值或回调改变时会重新构建。

这降低了受控 Slider 快速更新时 JS 端重复创建手势 builder 的频率。触摸移动计算、吸附逻辑、回调时机、手势阈值和动画 shared value 更新逻辑均保持原样。若调用方每次 render 都创建新的 `onValueChange` 回调，回调依赖会变化，手势仍会按依赖重建。

## 定量基准：Text 列表 React 协调

### 方法

- **运行时：** React 19.1、`react-reconciler` 0.31。
- **宿主环境：** 自定义内存 host renderer，用普通 JavaScript 对象替代真实 React Native 原生视图。
- **负载：** 挂载 120 个 props 不变的 Text 行，然后让父组件更新 30 次；每次更新都改变父级 revision，但行组件 props 保持不变。
- **重复次数：** 每个版本预热后测量 7 轮，报告中位数及观察到的范围。
- **树一致性：** 计算宿主树 SHA-256 前归一化父级 revision，避免测试变量污染树比较。

### 结果

| 指标 | 优化前 | 优化后 | 变化 |
|---|---:|---:|---:|
| Text 组件函数调用 | 3,720 | 120 | −96.8% |
| 原生文本宿主节点属性提交 | 3,630 | 30 | −99.2% |
| React 协调耗时中位数 | 17.59 ms | 7.95 ms | −54.8% |
| React 协调耗时范围 | 15.29–19.64 ms | 7.26–9.86 ms | 两组区间未重叠 |
| 归一化宿主树 SHA-256 | `b0d33a45bdb163161b4c276fb10522232351c12bd728204a44606563f922c1cf` | 相同 | 结构一致 |

变化率由表内原始计数计算：

- Text 调用减少：`1 − 120 / 3720 = 96.8%`
- 宿主属性提交减少：`1 − 30 / 3630 = 99.2%`
- 协调时间减少：`1 − 7.95 / 17.59 = 54.8%`

### 结果能说明什么

这个基准说明：对于 props 稳定、只因父组件更新而重复 render 的文本列表，memo 边界能减少 React 函数调用、子树协调和宿主属性提交。哈希一致说明这个基准中优化前后的渲染宿主树相同。

宿主 renderer 不包含真实原生布局、绘制、图片解码、GPU、设备触摸采样、Reanimated UI 线程或应用业务逻辑。因此 `54.8%` 是**该隔离负载下的 React 协调时间变化**，不能推导成整页、整应用或滚动 FPS 提升百分比。基准临时脚本在测量后已移除，仓库保存了运行参数和结果，但没有保留可直接复跑的基准脚本。

## 视觉检查

第一部分优化的 Release APK 安装到连接的小米 2405CPX3DC 设备后，对比了 onboarding 第 1/4 步的截图：

- 图像尺寸：`1224 × 2912`
- 比较范围：排除顶部 100 px 状态栏后，共 `3,441,888` 个像素
- 平均 RGB 绝对差：`0.0112 / 255`
- 变化像素：`5,540 / 3,441,888`，约 `0.16%`
- 最大单通道差异：`7 / 255`

肉眼检查显示页面位置、颜色和尺寸一致；少量像素差异符合截图与渲染波动。导航栏、滑块和当前选项动画没有在这次设备截图中被展示。本轮第二部分也没有重新执行设备截图或帧时间采集。

## 验证记录

- NewHu 根工程 `npm run typecheck`：通过。
- `project/react-native-hyper-ui` 中 `npm run typecheck`：通过。
- 组件库 `npm run build`：通过，生成 CommonJS、ES module 和 `.d.ts` 输出。
- 第一部分曾运行 `npm run android:release:arm64` 并安装 Release APK；本轮第二部分没有重新构建或安装 APK。
- 本报告没有把类型检查或构建当作性能测量，也没有为本轮添加 FPS 数值。

## 适用条件与局限

1. **Memo 只跳过 props 稳定的渲染。** 每次创建新的回调、对象、style 数组或 children 元素，会被浅比较视为变化。列表调用方应尽量复用稳定数据和回调。
2. **Context 更新仍会传播。** 主题或减少动效偏好改变时，消费这些 context 的控件必须更新，这是正确行为。
3. **原生和 GPU 开销未覆盖。** 图片加载、复杂文本布局、大型列表未虚拟化、原生阴影和主线程工作，需要在目标设备另行分析。
4. **没有第二轮设备 FPS 数据。** 本轮 Slider 手势稳定化和选项子树拆分有明确的减少重复对象 / 协调工作的机制，但尚无真机帧时间对照证明其对交互流畅度的实际幅度。
5. **兼容性范围由 peer dependencies 限定。** 本包针对 React Native 0.81 与 Reanimated 4 等依赖组合；性能结论不自动适用于其他 RN、Reanimated 或架构版本。

## 后续测量建议

要衡量用户实际感知的差异，下一步应在目标设备上使用相同 Release 构建、相同数据量和固定操作路径，对 Design System 展示页和包含大量列表项的页面采集 JS 线程与 UI 线程帧时间。测试应分别覆盖空闲父组件更新、Slider 连续拖动、底部导航切换和主题切换，记录中位数、P95/P99 帧时、掉帧数及设备型号；同时保留可复跑的 benchmark / profile 配置。
