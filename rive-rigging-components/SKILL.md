---
name: rive-rigging-components
description: Problem: Rive rigs and nested components fail in confusing ways. Bones don't deform artwork, IK knees flip, rigs jump after Create Component, nested events and inputs return null, nested animations freeze after one play, lists and data binding misbehave, and Hug/Fill or Fit.Layout sizing breaks, often only at runtime in rive-wasm, rive-react, rive-react-native, rive-flutter, rive-android or rive-ios. Solution: a symptom-indexed, ranked catalogue of these problems (as of Nov 2026) with documented fixes, workarounds, fix versions and evidence caveats, plus a triage procedure that separates editor from runtime causes and steers new work to data binding instead of deprecated events and inputs. Goal: get the user from a Rive symptom to a working, version-appropriate fix quickly, and help them build rigs and component hierarchies that avoid these problems in the first place. Use this skill whenever the user mentions Rive, a .riv file, a Rive state machine, rig, bones, mesh, weights, IK, joysticks, nested artboard, component, view model or data binding in Rive, or any Rive runtime bug, even if they don't say "rigging" or "nested".
---

# Rive rigging and nested components troubleshooting

This skill holds a ranked catalogue of Rive rigging and component problems with their fixes. The knowledge is current **as of late November 2026**, taken from Rive docs, community.rive.app, GitHub issues and release notes from Aug–Nov 2026. A lot of it is newer than the model's training data. Don't "correct" it from memory. Where the references are vague or flag something as unverified, keep it vague and point the user to the current Rive docs or release notes. Don't make up exact API names or version numbers.

## Two rules that apply to almost everything

1. **Update first.** Many reports from 2025 to early 2026 are fixed in Rive Editor 0.9.642+ (Nov 11, 2026) and in runtime releases from Oct 2026 or later. Before you dig deeper, ask which editor and runtime versions the user is on. If they are older than that, recommend updating, then retesting.
2. **Use data binding, not events or inputs, for new work.** Rive Events and state machine inputs are deprecated in the web, React Native and Apple runtimes. Rive does not plan to make events visible more than one level up. Do cross-artboard and app-to-file communication through view models. Older files can be migrated with Editor menu **Convert Inputs to ViewModels**.

## Triage procedure

0. **Find what actually renders it, and look at it moving.** Before giving advice, find the code or file that draws the thing the user means, on the surface they mean (often the deployed page). It may not be a Rive file at all: CSS, SVG or canvas layers have the same rigging problems. Don't base the work on a different file that merely looks related, such as an older rig or a draft artboard. If the user says something "looks odd", watch it at the extreme pose before you diagnose it.
1. **Classify the surface.**
   - **Editor or design problem** (it looks wrong in the Rive Editor). Usually a designer.
   - **Runtime problem** (it looks right in the editor but wrong in the app). Usually a developer.
   - **Both**, where the authoring choice is what causes the runtime failure. Events and inputs are the classic case.
2. **Collect the facts** you need. Ask only for what's missing:
   - Rive Editor version.
   - Runtime package and version, e.g. `@rive-app/canvas 2.38.3`, `rive 0.14.9` (Flutter), `rive-android 11.7.0`, `rive-ios v6.28.0`, `rive-react-native 9.3.4`.
   - Renderer: canvas, WebGL2, canvas-lite, Rust/Vello.
   - Nesting depth, and whether the problem involves lists.
   - Whether the file still uses Events or inputs.
3. **Look up the symptom** in the index below and read the linked section of the reference file.
4. **Answer in this shape:**
   - the likely cause, in one or two sentences;
   - numbered fix steps, using the exact editor menu names or API identifiers from the reference;
   - the version where the fix landed, if one is known;
   - a workaround in case they can't update;
   - any evidence caveat the reference attaches (unverified version, open issue, no official fix).
5. **When you change things, keep the scope tight and verify the way the user will look.** Touch only what was asked: don't edit side files, demo pages or other rigs along the way, and undo any such edits if they turn out to be wrong. Before saying it's fixed, check it in the real surface at the extreme pose, zoomed in, against the current live version (R11 *Verification*). A clean frame from your own test render isn't proof.
6. If nothing in the index matches, say so, give general guidance from the checklists, and suggest a minimal repro plus a check of the latest release notes. Don't make up a fix.

## Symptom index

### Rigging → [references/rigging.md](references/rigging.md)

| Symptom the user describes | Section |
|---|---|
| Rotating a bone doesn't bend the image or shape; it moves rigidly; bones invisible or can't be selected for binding | R1 Binding setup |
| Joints pinch or collapse; weight painting is tedious; all weights suddenly reset | R2 Weights |
| Knee or elbow flips; two IK constraints don't blend; FK/IK switching broken; constraint order; can't drag-reorder constraints; want look-at or physics | R3 IK and constraints |
| Rig jumps after Create Component, regrouping or reparenting; "These dependencies point across the new component's boundary and would break"; state machine missing in new component; bones over a component not clickable | R4 Rigs inside components |
| Joystick head turns / eye looks conflict; data-bound keyframes ignored with joysticks; driving poses from app data | R5 Joysticks and blend states |
| Skinned shape renders differently or vanishes on some runtime; fill winding wrong; mesh gaps (Rust); blank file on Android 11.7.0 | R6 Editor vs runtime rendering |
| Rig is slow on mobile; vertex budgets; frame time | R7 Performance |
| Solo bounds wrong; blank page in a Solo of Components; draw-order problems | R8 Solos and draw order |
| Gradient doesn't follow a bone-bound shape | R9 Gradients |
| Importing a rig from Spine, After Effects/Lottie, Animate (.fla), Moho or Live2D | R10 Migration |
| An ear, tail or other part cut from one flat image looks odd when it moves; gap, wedge or sliver at the joint; markings shear; "ear movement" | R11 Moving parts cut from one flat image |
| "Review my rig" or "best practices" | Rigging checklist |

### Components, events, layouts → [references/components.md](references/components.md)

| Symptom the user describes | Section |
|---|---|
| Event from a deeply nested component isn't heard by the grandparent; Rive Event deprecation warnings; Open URL events | C1 Events across levels |
| `useStateMachineInput` returns null; nested controller has 0 inputs; "how do I call a nested artboard input"; exposed inputs not working | C2 Nested inputs |
| Nested animation plays once and then freezes or won't replay; instances all start in sync; random values don't regenerate | C3 Nested playback |
| List shows one item or renders erratically; nested values stop updating; empty instance dropdown; view model renamed number→number2 | C4 View model wiring |
| Works in the editor but nested components or lists don't render at runtime; `autoBind`; pointer events dead in nested lists; bound boolean doesn't start until touched | C5 Runtime data binding |
| Hug reverts to fill on reopen; gaps in grid rows; list items can't fill width; stale text wrapping in a Solo | C6 Hug/Fill sizing |
| `Fit.Layout` doesn't resize, or resizes late; surface stuck at creation size; invisible when responsive | C7 Fit.Layout |
| Pointer listeners in components don't fire; scroll issues; capture listener blocks children | C8 Listeners and scrolling |
| Text runs, audio or script sounds inside components | C9 Text, audio, content |
| "Is this a component?"; artboard not exported; component setup basics | C0 Terminology (top of file) |
| "Review my file" or "best practices" | Components checklist |

### Runtime APIs, versions, breaking changes → [references/runtime-and-versions.md](references/runtime-and-versions.md)

Read this file when the question is about a specific runtime API, an upgrade, a deprecation warning, or "which version fixed X". It also covers:
- timeline names no longer exported by default (0.9.642);
- the Apple `viewModelInstance` returning nil (v6.28.0);
- the web `stateMachines` deprecation (v2.41.0);
- React Native v0.5.0 changes.

## Evidence grades: keep them visible

The references tag some claims. Pass these caveats on to the user rather than stating the claim as settled fact:
- **[mirror]**: editor version numbers 0.9.415–0.9.708 come from the Releasebot aggregator mirroring Rive's announcements. They agree with official sources where the two overlap, but they haven't been checked against the original posts.
- **[unofficial]**: rules of thumb from secondary sources, such as the <500 / <5,000 vertex budgets.
- **[open]**: the issue was still unresolved as of Nov 2026, e.g. rive-wasm #448.
- **[no fix found]**: no official reply or fix was located, e.g. the weight-reset bug.
- **[unclear]**: the source didn't show the final steps, e.g. the gradient-binding thread.

Rankings inside each file reflect how visible and how severe the public reports are, not usage telemetry. Discord could not be searched.
