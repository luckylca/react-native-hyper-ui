# Hyper UI performance optimization report

## Scope

The optimization targets repeated React rendering in reusable controls and dense slider key points. It keeps the current component API, theme tokens, layout values, gesture definitions, Reanimated worklets, spring/timing configurations, and rendered native host tree intact.

## Changes

- Added shallow prop memoization to `Text`, `Icon`, `Surface`, `Divider`, `ListRow`, `Button`, `Card`, `Switch`, and `Slider`. Stable props now skip function execution and native child reconciliation. Context changes still update theme consumers, and changed props still render normally.
- Stabilized the empty slider key-point list, memoized derived key points, and made each marker a memoized leaf. When a controlled slider value changes, markers whose position and active state are unchanged can skip rendering.
- Did not change animation code or visual tokens. In particular, press indication, card sink/tilt, switch thumb/track springs, slider movement, and sheet/dialog transitions keep their existing implementation and parameters.

## React reconciliation benchmark

I compared the same `Text` implementation with and without the new memo boundary using React 19.1 / `react-reconciler` 0.31. The host renderer substitutes native views with in-memory host nodes; it measures React work and host prop commits, not GPU frame time.

Workload: 120 stable text rows; mount once, then update the parent 30 times while every row keeps the same props. Seven measured rounds per variant followed a warm-up; the table reports median elapsed time. Output host trees were hashed after normalizing away the changing parent revision.

| Metric | Before | After | Change |
|---|---:|---:|---:|
| Text component renders | 3,720 | 120 | −96.8% |
| Native text host prop commits | 3,630 | 30 | −99.2% |
| Total reconciliation time | 17.59 ms | 7.95 ms | −54.8% |
| Rendered host tree SHA-256 | `b0d33a45bdb163161b4c276fb10522232351c12bd728204a44606563f922c1cf` | same | identical |

The timing range was 15.29–19.64 ms before and 7.26–9.86 ms after. The result supports reduced repeated JS reconciliation for stable list content. It does not predict the same percentage improvement in a full app, where native layout, image loading, and application state updates also consume time.

## Device visual check

Built and installed the optimized Release APK on the connected Xiaomi 2405CPX3DC. The app was at onboarding step 1/4 before and after installation. I compared screenshots at 1224 × 2912, excluding the top 100 px system status area: mean absolute RGB difference was 0.0112 on a 0–255 scale, with 5,540 changed pixels out of 3,441,888 (0.16%) and maximum channel difference 7. Visual inspection showed the app content kept the same placement, colors, and dimensions. Remaining pixel noise is within small screenshot/rendering variation.

The app deep link landed on onboarding, so this device check did not reach the design-system showcase or capture a representative list-scroll FPS trace. Animation worklet and preset files were not changed; this is source-level evidence that the motion parameters are preserved, not a measured animation frame-time claim.

## Validation

- `npm run typecheck` passed in the app root.
- `npm run typecheck` passed in `project/react-native-hyper-ui`.
- `npm run build` passed in `project/react-native-hyper-ui` (CommonJS, ES module, and declaration output generated).
- `npm run android:release:arm64` completed; Release APK installed successfully.
