# Rive nested artboards (Components), events and layouts: ranked issues and fixes (as of Nov 2026)

Tags: **[mirror]** means the version number comes from the Releasebot aggregator and hasn't been checked against the original post. **[open]** means still unresolved as of Nov 2026. For runtime API details and breaking changes, see runtime-and-versions.md.

Contents: C0 Terminology · C1 Events across levels · C2 Nested inputs · C3 Nested playback · C4 View model wiring · C5 Runtime data binding · C6 Hug/Fill sizing · C7 Fit.Layout · C8 Listeners and scrolling · C9 Text, audio, content · Components checklist

---

## C0. Terminology (changed in 2026)

- Nested artboards are now called **Components**.
- The Nested Artboard Tool is now the **Component Tool** (shortcut **N**).
- An artboard must be **marked as a component** (Shift+N) before it can be placed.
- **Artboards that aren't components are not exported** (Components docs). If an artboard is "missing at runtime", check this first.

## C1. Events don't reach past one level; Rive Events are deprecated

**Symptoms**
- An event fired in "Source" is heard in "Artboard 1", which contains it, but not in "Artboard 2", which contains Artboard 1. Feature request: "Expose component events across multiple nested artboards".
- Rive staff, in a bug thread linked from rive-ios #303: "we don't have any plans to expose these. An event can only be observed on level deep and you'll need to 'bubble' up the event."
- The Listeners docs: "We strongly recommend using Data Binding to communicate between artboards instead of relying on nested Rive events."

**Deprecations**
- **Web:** as of JS runtime v2.41.0, these are deprecated. Each logs a one-time console warning; nothing has been removed yet.
  - `EventType.RiveEvent`
  - `EventType.StateChange`
  - `EventType.Loop`
  - `automaticallyHandleEvents`, which handled Open URL events
- **React Native:** inputs, text runs and events are deprecated, and "will be removed in the experimental runtime."
- **Apple:** the new runtime exposes no input APIs. The migration guide says to model app-facing signals as data-binding properties.

**Who:** both. Designers build the bubbling, and developers lose the event callbacks.

**Fix**
1. **Preferred:** put a trigger or boolean on a view model shared by parent and child. The child's state machine sets it, and the parent's listener or transition (or the app) reacts. No extra wiring is needed per nesting level.
2. **Legacy bubbling:** in each intermediate artboard, add a listener with the nested component as **Target** and **Listen to** = Rive Event. Its action fires a new event of its own. This passes the event up one level at a time.
   - Only artboards or nested components can be targets for events.
   - Artboard targets support custom events, not pointer events.
3. **Open URL:** have the app observe a view model trigger or string and open the URL itself.
4. **Migrating old files:** Editor menu → **Convert Inputs to ViewModels**.

**Status:** by design, and not planned. Data binding is the official replacement.

## C2. Nested inputs unreachable from code, or returning null

**Symptoms**
- rive-react #330 (Feb 2025): `useStateMachineInput` returned null for both the parent and the child, even with "Expose to parent artboard" set.
- Similar reports:
  - rive-flutter #341: the nested controller had 0 inputs.
  - rive-react-native #242: "How do I call a nested artboard input".
  - rive-ios #303: "exposed inputs from nested artboards are not working".

**Who:** developers.

**Fix**
1. **Use the path-based APIs.**
   - **Flutter 0.13.6+:** `getBoolInput` / `getNumberInput` / `getTriggerInput(name, path)`.
   - **Flutter 0.14:** `stateMachine.boolean('myBool', path: 'nested/path')`.
   - **Web:** the `set…AtPath` family and `fireStateAtPath(inputName, path)`. The source only names them as "set...AtPath"; check the web runtime docs for the exact method names before quoting them.
   - **Text runs:** `setText(name, value, path)`.
   - **The path uses the nested artboard's *instance* name, not its source artboard name.** Use slashes for deeper levels.
2. **Name the artboard explicitly.** rive-react #330 noted that the default artboard can resolve differently than expected.
3. **Better: migrate to data binding with nested property paths.**
   - Flutter 0.14.0-dev.14 marked the input and text run methods deprecated.
   - The web docs say "Prefer data binding with Nested Property Paths for nested artboard properties."

**Status:** path-based access works but is legacy. Data binding is the supported path.

## C3. Nested animations play once, then freeze or won't restart

**Symptoms**
- "Solved: When using nested artboards in Rive, the animations inside them play correctly the first time, but they do not loop properly when triggered again from the main (parent) artboard."
- "Issue with Nested Artboard Animation Not Replaying."
- Older feedback item #711: a One Shot nested animation wouldn't replay on state re-entry. Staff called it a bug.

**Who:** designers.

**Cause**
- A component instance can run a **State Machine**, or timeline animations in **Simple** or **Remap** mode (Components docs).
- A finished **Simple** timeline doesn't rewind when the parent re-enters a state. As a Medium write-up put it: "once an animation is played, it doesn't ever start back at the beginning… unless it is specifically called to do that."

**Fix**
1. Give the child artboard its own state machine and select it on the instance. Drive it with a view model trigger or boolean from the parent, so each trigger replays.
2. Or add the timeline in **Remap** mode and key **Time** from 0% to 100% in the parent's timelines. Each parent state then controls the playhead directly.
3. If the child state machine waits at Entry, start it from **Any State** with a condition (Viget's tutorial fix).
4. To stagger identical instances, use separate state machines, or Remap with offsets (forum thread "Multiple nested artboards start playing at different times").
5. Random values generated "only on state machine start" don't regenerate in nested artboards driven only by the parent's animation. Run the child's state machine on the instance.

**Status:** consistent with how Simple and Remap work; the forum threads are marked solved. Editor 0.9.642 and CLI 1.9.2 also fixed a related runtime case: "a transition with early exit no longer jumps when it interrupts a blend."

## C4. View model wiring mistakes across parent, child and list items

**Symptoms**
- "Solved: Data binding list issue": a list showed one item or rendered erratically. The child artboards (icons, arrows, scoreboard) had been assigned the same "widgetModel" view model as the list item. Removing the view model from the children fixed it.
- "I SOLVED IT: How to use Components like 'Pre-Comps'": a nested component's values stopped updating after it was moved out of the parent's hierarchy.
- An empty instance dropdown for nested artboards has the same root cause.

**Who:** designers first, then developers.

**Fix**
1. Give **each component its own view model**.
2. In the parent's view model, add a **View Model property** whose type is the child's view model.
3. Select the component instance and set **Data > Model** to that property. Alternatively, use **Inherit**, which "uses a compatible view model instance from the closest parent."
4. Don't assign the parent's view model to child artboards inside a list item.
5. For list rows, bind the list to a list property whose items are instances of the row's view model.

**Editor fixes in 0.9.642** (Nov 11, 2026)
- Duplicating a view model no longer renames every property (number → number2), which had broken code that reads values by name.
- Picking a different instance for a view model property now updates the components bound through it.
- Lists whose rows contain the same list no longer crash the editor on relaunch. Rive now stops building those rows and shows a message.

## C5. Runtime data binding of nested components and lists fails while the editor looks right

**Reports**
- **rive-wasm #448 [open]** (opened Nov 10, 2026): with `autoBind: true` on `@rive-app/canvas 2.38.3`, three character components bound to the root view model stop rendering. They show correctly in the editor.
- **rive-react #410** (Mar 2026): nested lists populated by a script didn't render in react-webgl2. The maintainer suggested `autoBind: true` or binding the instance manually. The issue was closed Nov 5, 2026 for lack of response.
- **rive-flutter #644** (Jul 2026, rive 0.14.9): a component supplied through an artboard property and containing an `ArtboardComponentList` never got pointer events when the slot assignment and the list population ran in the same batch. A Rive engineer replied (Aug 3, 2026): "this sounds like we're not handling state machine settling correctly for nested components that are supplied through binding."
- **rive-react-native #348** (Jun 2025, 9.3.4): a data-bound boolean didn't start the timeline until the user touched the artboard.
- **rive-nitro-react-native discussion #124** (Jan 2026): `useViewModelInstance` ignored the view model name when given a `riveFile`. A maintainer confirmed the bug.

**Who:** developers.

**Fix**
1. **Bind explicitly.**
   1. Get the view model by name.
   2. Create the instance.
   3. Call `bindViewModelInstance`.
   4. Make sure the state machine advances.

   Two runtimes now bind at construction:
   - Apple v6.28.0 binds authored default instances automatically.
   - Flutter 0.15.0 binds when `RiveWidgetController` is created.
2. **Advance after structural changes.** After changing artboard properties or lists from timers rather than input callbacks, advance the state machine (e.g. by zero time) so hit-testing settles. The #644 reporter found that changes made inside a real pointer callback always converged.
3. **Update runtimes.** Fixes in 2026 release notes:
   - **rive-android:**
     - List-item state machines always used the first state machine, even when another was the default.
     - Nested image and font bindings through exposed component properties failed to update.
     - Script-hosted triggers stopped after their first activation.
   - **Core** (Oct 1, 2026):
     - Interpolators no longer restart on a view model re-bind.
     - Releasing a list child keeps the parent's subscriptions.
     - Hidden components and lists no longer swallow pointer hits.
   - **rive-ios:** supports binding list-item content to nested view model values from the host.

**Caveat:** whether #448 is fixed in any particular release is **not confirmed**. Tell the user it was still open in Nov 2026.

## C6. Hug/Fill sizing of components and list items is unreliable

**Symptoms**
- "Solved: Hug feature not working on nested artboard layouts" (Jul 2025): Hug around nested list items reverted to Fill each time the file was reopened. Workaround: hide and re-show the layout.
- "Solved: Rows set to Hug do not hug resized nested artboards": gaps appeared between grid rows.
- "[Data Binding List] How to Enable 'Fill Width'…": list item artboards offered only Fixed and Hug.
- "Unable to select 'Hug' for Child Layout."
- "Nested component text wrapping issue": text inside a Solo switched by an enum kept stale wrapping.

**Who:** designers building responsive UI.

**Fixes, by version**
- **Runtime roundup, Aug 31, 2026:** "Components honor fill and hug inside grid and stack layouts, pick the right axis when a Solo or group separates them from their parent layout, and keep their origin when a constraint applies." Text and shapes inside Solos now inherit layout size.
- **rive-android** has matching fixes:
  - fill/hug in grids and stacks;
  - constraints discarding a component's origin offset;
  - wrong placement inside rotated or scaled layouts;
  - paused components missing layout changes.
- **Editor 0.9.479:** Hug layouts measure their contents fewer times.
- **Editor 0.9.517:** adds an Aspect Ratio setting for layouts and fixes dragging in right-to-left layouts.

**Workarounds that still apply**
1. Set the component instance's sizing to **Hug** in the parent layout. With Fixed or Fill, the parent ignores size changes driven by the child's state machine (Rive 101, reflow lesson).
2. **List items that fill the width:** put the fill behavior on a layout *inside* the item artboard, and set the item instance's **Mode = Layout** (Component Sizing docs).
3. **Layout reverts after reopening:** toggle its visibility once, and update the editor.

## C7. Fit.Layout at runtime doesn't resize, or resizes late

**Symptoms**
- **rive-android #446** (11.3.2, originally rive-nitro-react-native #206): `Fit.LAYOUT` gave the wrong artboard size if `setRiveFile` ran before `onMeasure`, or if the state machine keyed the artboard width.
- **rive-nitro-react-native #418:** on the new Android backend, the render surface kept its creation size after a resize. This often happens when a keyboard animates in.
- **rive-react #409** (Mar 2026): `Fit.Layout` didn't reflow in canvas-lite.
- Forum thread "Problem with Responsive Layout" (Jan 2026): the animation became invisible once the canvas was made responsive.

**Who:** developers.

**Fix**
1. Build the artboard with Layouts and set `fit` to Layout. The default is Contain. Content outside a Layout "won't respond to the new size" (Artboards docs).
2. **Web:** call `resizeDrawingSurfaceToCanvas()` on load, on window resize, and when `devicePixelRatio` changes. With Fit.Layout it also resizes the artboard.
3. **Android legacy view:** set `layoutScaleFactor` (e.g. to the screen density) **before** `setRiveFile`.
   - Rive PR #450 resizes the artboard every frame under `Fit.LAYOUT`.
   - Nitro PR #419 (Oct 13, 2026) resizes the render surface when the view size changes.
4. Don't key the artboard's own width or height in a state machine if you rely on Fit.Layout.
5. **Flutter:** the legacy runtime didn't support Fit.Layout. Use 0.14.0+, which added layouts, scrolling and N-slicing.

## C8. Listeners, hit areas and scrolling inside components

**Symptoms**
- Pointer listeners in nested components don't fire, or the wrong layer catches the press.
- Scroll constraints ignore mouse wheels, fling content that already fits, or swallow taps.
- Two-finger drags make scroll views jump.

**Fixes, by version**
- **Core runtime** (Oct 1, 2026):
  - `ScrollConstraint` responds to mouse wheels and trackpads.
  - Content that fits no longer flings.
  - A pixel of wobble no longer cancels a tap.
  - Hidden components and lists no longer take pointer hits.
- **Editor 0.9.642:** the newest finger drives two-finger scrolling. Hover clears on list rows that scroll out of view.
- **Editor 0.9.708** (Nov 18, 2026) **[mirror]**:
  - A listener with **Phase = Capture** no longer blocks presses to the components and Text Inputs inside it.
  - A new **Stop Propagation** checkbox restores the old behavior.
  - Runtimes adopt this as they update.
- **Opaque Target** (docs) decides whether pointer events stop at a listener's hit area or pass through to listeners behind it.

**Who:** both.

**Workarounds**
- Turn Opaque Target on or off deliberately.
- Give components an explicit hit shape.
- Retest capture listeners after updating to 0.9.708.

## C9. Text, audio and other content inside components

**Text runs**
- Text runs in nested artboards used to need path-based access (`setText(name, value, path)`).
- That approach is now deprecated. Use data binding.

**Audio**
- **Editor 0.9.517:** audio stopped with a fade stayed silent when played again. Fixed: "Playing it now starts it from the beginning."
- **Editor 0.9.708 [mirror]:** script-started sounds kept playing after their artboard was removed. They now stop with the artboard when started in init, advance or input code.

**Who:** both.

---

## Components, events and layouts checklist

1. Mark reusable artboards as Components. Unmarked artboards are not exported.
2. Give every component its own view model. Connect it through a View Model property on the parent, and set the instance's Data > Model to it, or use Inherit.
3. Don't rely on Rive Events or inputs for new work. Use view model triggers and booleans. Convert old files with Convert Inputs to ViewModels.
4. Give nested animations a state machine on the instance, or use Remap. Don't count on Simple timelines replaying.
5. At runtime, bind the view model instance explicitly. Advance the state machine after structural changes made outside input callbacks.
6. Set component instances to Hug inside layouts that must reflow. Put fill behavior inside the item artboard's own layout.
7. Use Fit.Layout only with layout-built artboards. On the web, call `resizeDrawingSurfaceToCanvas`. On Android, set `layoutScaleFactor` before loading.
8. Turn on **Export name** for any timeline your code plays by name (Editor 0.9.642+).
9. Retest listeners after 0.9.708's change to capture listeners.
10. Before filing a bug, update to Editor 0.9.642+ and runtime releases from Oct 2026 or later. Many 2025-era reports are already fixed there.
