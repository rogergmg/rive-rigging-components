# Rive runtimes: APIs, breaking changes and fix versions (as of Nov 2026)

Use this file for upgrade questions, deprecation warnings, and "which version fixed X". Things change quickly, so tell the user to check the current release notes before relying on an exact version. **[mirror]** marks editor versions that come from the Releasebot aggregator and haven't been checked against Rive's original announcement.

## Breaking changes to watch (late 2026)

### Rive Editor 0.9.642 (Nov 11, 2026)
- **Timeline names are no longer exported by default** in new files. If code plays a timeline by name, turn on **Export name** for that timeline, or use **Export all animation names**.
- State machine names still export.
- State machine *layer* names no longer export.

### Apple runtime v6.28.0
- With the new initializers, `Rive.viewModelInstance` returns **nil**.
- Deprecated:
  - `DataBind`
  - `dirtyStream()`
  - singular `bindViewModelInstance(_:)`
- `Rive(file:dataBind: .auto)` keeps the old behavior for now.
- Authored default instances are bound automatically.
- The new runtime exposes no input APIs. Model app-facing signals as data-binding properties.

### Web runtime (JS) v2.41.0
- The plural `stateMachines` parameter is deprecated in favor of `stateMachine`.
- In v3, the artboard's default state machine will play when none is named.
- Deprecated, each with a one-time console warning (nothing removed yet):
  - `EventType.RiveEvent`
  - `EventType.StateChange`
  - `EventType.Loop`
  - `automaticallyHandleEvents`

### React Native
- v0.5.0 is out of beta as of the Oct 1, 2026 roundup.
- It moved to the newer iOS and Android runtimes.
- `RiveAsset` became `RiveFileSource`.
- Inputs, text runs and events are deprecated, and "will be removed in the experimental runtime."

### Flutter
- **0.13.6+:** path-based input getters, `getBoolInput/getNumberInput/getTriggerInput(name, path)`.
- **0.14.x:** `stateMachine.boolean('myBool', path: 'nested/path')`. Adds layouts (`Fit.Layout`), scrolling and N-slicing; the legacy runtime had no Fit.Layout.
- **0.14.0-dev.14:** input and text run methods marked deprecated.
- **0.15.0:** binds when `RiveWidgetController` is created.

## Data binding at runtime: the generic recipe

1. Load the file and name the artboard explicitly. Don't rely on the default artboard (rive-react #330).
2. Get the view model by name, create an instance (or use the authored default), and bind it with `bindViewModelInstance`, or use the runtime's auto-bind (`autoBind: true` on web).
3. Make sure the state machine advances after binding.
4. When you change lists or artboard properties outside an input callback, advance the state machine (e.g. by zero time) so layout and hit-testing settle (rive-flutter #644).
5. For nested components, use nested property paths rather than path-based inputs.

Exact method names differ per runtime and change between versions. Check the runtime's current data binding docs before you give exact signatures beyond the ones listed here.

## Fit.Layout per platform

| Platform | What to do |
|---|---|
| Web | `fit: Layout`; call `resizeDrawingSurfaceToCanvas()` on load, on resize, and when `devicePixelRatio` changes. canvas-lite had a reflow bug (rive-react #409). |
| Android (legacy view) | Set `layoutScaleFactor` before `setRiveFile`. PR #450 resizes the artboard every frame under `Fit.LAYOUT` (fixes #446). |
| React Native (nitro, new Android backend) | PR #419 (Oct 13, 2026) resizes the render surface on view size changes (#418). |
| Flutter | Requires 0.14.0+. |
| All | Don't key the artboard's own width or height in a state machine. Content outside a Layout won't resize. |

## Known issues and regressions by runtime

### rive-wasm / @rive-app/canvas
- **#448 [open]:** with `autoBind: true` on 2.38.3, three character components bound to the root view model stop rendering.
- **PR #14284:** fixed partial-strength IK blending (Sep 2026).

### rive-react
- **#330:** `useStateMachineInput` returned null for nested inputs.
- **#409:** Fit.Layout doesn't reflow in canvas-lite.
- **#410:** script-populated nested lists don't render in webgl2. Closed without resolution.

### rive-react-native / nitro
- **#242:** "how to call a nested input".
- **#348:** a bound boolean doesn't start until touched (9.3.4).
- **#206 → android #446:** Fit.LAYOUT sizing.
- **#418:** surface stuck at creation size.
- **Discussion #124:** `useViewModelInstance` ignores the view model name with a `riveFile`. Confirmed bug.

### rive-flutter
- **#341:** nested controller has 0 inputs.
- **#644:** no pointer events in a binding-supplied component containing an `ArtboardComponentList` (0.14.9). Rive engineer: a state-machine settling issue.

### rive-android
- **11.7.0 regression (#459):** blank render; 11.6.1 worked. Pin a version if needed.
- Release-note fixes:
  - list-item state machines;
  - nested image and font bindings;
  - script triggers that fired only once;
  - fill/hug in grids and stacks;
  - origin offset;
  - rotated or scaled layouts;
  - paused components.
- Data-bound keyframes with joysticks.

### rive-ios
- **#303:** exposed nested inputs don't work.
- Release-note fixes (exact release tag not seen):
  - data-bound keyframes ignored by joystick-driven, nested and standalone animations;
  - binding list-item content to nested view model values from the host.

### rive-rs (Rust, Vello)
- Known mesh issues: gaps at triangle borders, and transparent meshes overdraw at the borders.

## Oct 1, 2026 runtime roundup (core): highlights

**Rigging**
- IK at partial strength no longer wipes out a second constraint's solve.
- A vertex with no weights keeps its bind position.
- Malformed rig data fails to load instead of crashing.
- Winding on bone-bound paths now comes from the deformed shape.
- Skinned fills no longer vanish after loading folded.
- Hidden paints re-run their effects when shown.

**Performance**
- Fades stop re-skinning.
- Rigid moves keep their path geometry.
- Blend states set each property once.
- Cheaper updates for many hidden shapes.
- New `drawImageMeshInstanced`.

**Data binding**
- Interpolators no longer restart on re-bind.
- Releasing a list child keeps the parent's subscriptions.
- Hidden components and lists don't take pointer hits.

**Scrolling**
- Responds to wheel and trackpad.
- Content that fits doesn't fling.
- A pixel of wobble doesn't cancel a tap.

## Editor build timeline

| Build | Date | Notable |
|---|---|---|
| 0.9.415 [mirror] | fall 2026 | Solo switches don't re-lay out the artboard; large rigs without layouts ~35% faster |
| 0.9.460 | Oct 30, 2026 | Pointers follow mesh/N-Slice deformation on Components; mirrored-bone fill direction |
| 0.9.479 | Nov 1, 2026 | Formula converter speedup (0.18 → 0.035 ms); Hug measures less |
| 0.9.517 | Nov 3, 2026 | Bitmap Cache off → 1x mesh draw; WebGL2 clipping; layout Aspect Ratio; RTL dragging; audio replay |
| 0.9.642 | Nov 11, 2026 | Constraint drag-reorder; Create Component Keep on Instance / Move into Component; Offset Keys; Solo fixes; view model duplicate/rename fixes; recursive list crash; two-finger scroll; **timeline names not exported by default** |
| 0.9.708 [mirror] | Nov 18, 2026 | Capture listeners no longer block children, plus a Stop Propagation checkbox; script sounds stop with their artboard |

Other dates: the runtime roundup of Aug 31, 2026 (layout and Hug fixes); Rive CLI 1.9.1 out of technical preview (Nov 2026); CLI 1.9.2 (early-exit transition fix); .fla import (Oct 2026).

The 0.9.642 date is cited in several places, but all the 0.9.415–0.9.708 numbers trace back to the Releasebot mirror. They agree with the official Oct 1 roundup where the two overlap. Treat exact build numbers as "approximately this release" and confirm them against Rive's announcements if the user needs precision.
