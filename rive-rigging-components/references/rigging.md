# Rive rigging: ranked issues and fixes (as of Nov 2026)

Tags: **[mirror]** means the version number comes from the Releasebot aggregator and has not been checked against Rive's original post. **[unofficial]** marks a secondary-source heuristic. **[no fix found]** and **[unclear]** are explained in SKILL.md.

Contents: R1 Binding setup · R2 Weights · R3 IK and constraints · R4 Rigs inside components · R5 Joysticks and blend states · R6 Editor vs runtime rendering · R7 Performance · R8 Solos and draw order · R9 Gradients · R10 Migration · R11 Moving parts cut from one flat image · Rigging checklist

---

## R1. Bones won't deform the artwork (binding setup errors)

**Symptoms**
- A bone rotates but the image or shape doesn't bend, or it moves rigidly. Typical forum thread: "Newbie question: binding bones + mesh won't deform raster".
- Bones show in the hierarchy but are invisible on stage, so they can't be picked for binding. Thread: "Bones disappearing from artboard right after creation and unable to select them for binding".

**Who:** designers, mostly beginners.

**Cause:** there are three different bone mechanisms, and they are easy to confuse (Bones docs):
- *Parenting* an element to a bone moves it rigidly.
- *Binding* a vector path's vertices and Bézier handles deforms the path.
- *Deforming a raster image* needs a **mesh** first, then binding of the mesh's vertices.

Parametric shapes such as rectangles and ellipses must first be converted to a custom path: select the path layer and press Enter. The Flare-era manual says the same: "select a path layer in your hierarchy, not a shape layer."

**Fix**
1. **Image:** add a Mesh (Deform section, +) and edit the contour so there are vertices near each joint.
2. **Vector:** convert the shape to a custom path. Add vertices where it should bend. The old manual suggests "three on each side so it bends smoothly", and warns that more vertices make weighting harder.
3. **Bind:** select the image or path and click + in Bind Bones. Hold Cmd+Shift (macOS) or Ctrl+Shift (Windows), select the bones, then click Done. The bones and the vertex pie charts take on matching colors.
4. **Invisible bones:** open View Options and turn Bones on.
5. **Component instance:** one with Mode = Node can take a mesh (Meshes docs). "The component draws its artboard as a flat image and the mesh deforms that image, so you can bind it to bones and animate its vertices like any other mesh."

**Status:** working as designed. The docs were rewritten in 2026 with videos and the "Mesh and Bones" Marketplace remix files.

## R2. Weight painting is tedious, and weights can get lost

**Symptoms**
- Pinching or collapse at joints.
- Weights have to be set vertex by vertex.
- A 2026 bug report, "Resetting weights from all meshes without reason", says every weight on every mesh was reset, and "Reverting with Ctrl+Z didn't help." **[no fix found]**: no official reply or fix was located.

**Who:** designers and animators.

**Rules and tools** (Bones docs)
- A vertex's weights always total 100%. Rive assigns initial weights automatically.
- **Edit Weights:** select vertices and a bone, then drag up or down to change that bone's influence.
- **Smooth:** evens out weights across neighboring vertices (the selection, or the whole shape if nothing is selected).
- **Auto-Weights:** reassigns weights by nearest bone.
  - **Blend** sets the width of the boundary blend; higher values give softer bends.
  - **Influence** sets the maximum number of bones per vertex.
- **Lock** a bone's weight on a vertex so later edits don't change it.
- On vector paths, Bézier handles can be weighted separately from their vertex.
- **Show Weights** toggles the pie charts. The mesh view's **Deform** toggle switches between the deformed and undeformed image.

**Workarounds**
- Run Auto-Weights first, then Smooth only at the joints.
- Weight joint vertices about 50/50 between neighboring bones, and 75/25 one ring away (old manual's recipe).
- Lock finished weights before working on neighboring bones.
- On vectors, raise the vertex Radius instead of adding vertices (old manual).
- **Save a version or duplicate the file before large weight edits.** Undo didn't recover the reported weight loss.

**Status:** tooling has improved (Auto-Weights with Blend/Influence, weight locking). The Oct 1, 2026 runtime roundup also has related fixes:
- "a vertex with no weights keeps its bind position";
- malformed rigging data now fails to load instead of crashing.

## R3. IK bends the wrong way, stacked IK cancels, constraint order matters

**Symptoms**
- Knees or elbows flip.
- Two IK constraints on one chain don't blend, for example when switching knee direction.
- One user ("Solved: IK leg boning", date unknown) reported "so many weird twitches and unnatural movements, especially when nesting artboards".

**Who:** designers who rig characters, and developers when the runtime result differs from the editor.

**Documented behavior** (IK Constraint docs)
- The constraint goes on the **last bone** of the chain. **Bone Count** sets how far up the chain it reaches.
- **Invert Direction** swaps the solve angle. It can be keyed in Animate mode.
- **Order matters:** when two IK constraints are both at 100%, the lower one overrides the upper. Below 100% they blend.
- **Strength** can be keyed, which gives an FK↔IK blend.

**Runtime bug, now fixed**
- Commit "fix: blend IK from the pose on its chain, not the bones' authored angles" (rive-wasm PR #14284, Sep 2026).
- What went wrong: an IK below full strength reverted its chain to the authored angles. A knee flip cross-faded by keying strength 0/1 therefore never bent the leg, and rotation constraints higher up the chain read back 0.
- The Oct 1, 2026 roundup lists it as "IK at partial strength fixed: a second constraint on the same chain no longer wipes out the first one's solve."

**Fix**
1. Put the IK on the last bone. Set Bone Count = 2 for a leg or arm. If a hip-knee-heel chain twists, move the foot and toes into a separate bone group (from the "IK leg boning" answer).
2. Set the bend side with Invert Direction, and key it if the limb has to switch sides.
3. For a limb with two possible bends, use two IK constraints and key their Strength 0/1. This is only reliable on runtimes built after the Sep–Oct 2026 fix, so re-export and update the runtime.
4. Reorder constraints by dragging them in the Inspector. Dragging was broken for a while and was restored in **Editor 0.9.642** (Nov 11, 2026): "dragging a constraint in the Inspector's Constraints section reorders it again."
5. Translation Constraints on root bones gained an **Offset** option in a fall 2026 editor release. **[mirror; exact version not confirmed]**

**Missing features**
- No aim / look-at constraint and no stretch-to option. Feature request: "Look At Constraint".
- **No physics constraint.** From Rive's Rive vs Spine page: "you add physics like gravity and springs with Scripting."

## R4. Rigs jump, break or stop updating inside components

**Symptoms:** a rig works on its own artboard but misbehaves once nested.
- Keyed root bones jump after Create Component.
- Animations jump after regrouping or reparenting.
- From an old changelog: "distance constraint changes in a nested artboard [were] not updating in the parent artboard."
- Bones drawn over a Component can't be clicked on stage.

**Who:** designers who build a character once and reuse it.

**Fixes, by version**
- **Editor 0.9.642** (Nov 11, 2026):
  - Create Component now asks whether keyed transforms stay on the instance (**Keep on Instance**) or move inside it (**Move into Component**). It "no longer makes keyed root bones jump."
  - Group, Ungroup or Set Parent on an animated object now offers **Offset Keys**, which adjusts position keys, curve handles and constraint limits to the new parent.
- **Fall 2026 release:** bones drawn over a Component are clickable again. **[mirror; version not confirmed]**
- **Editor 0.9.460** (Oct 30, 2026): pointers on a Component drawn through a mesh or N-Slice now follow the deformation.

**Create Component limitations** (docs troubleshooting)
- It fails with **"These dependencies point across the new component's boundary and would break"** when, for example, a draw-order dependency crosses the selection.
- State machines are not copied into a new component.
- View models are not copied either; only their bindings are.

**Workarounds**
1. Rig inside the artboard that will become the component, not in the parent.
2. Before converting, include every constraint target, draw-order rule and clipping shape in the selection.
3. Rebuild the state machine inside the new component, and drive it from the parent through data binding (components.md C1/C4).
4. A component's origin can't be changed directly. Wrap the instance in a group and move the group's origin (Components docs).

## R5. Joysticks and blend states for head turns and poses

**Context**
- A joystick is one handle that scrubs an X timeline and a Y timeline. It is Rive's standard way to build 2.5D head turns and eye looks.
- Examples: Rive's crocodile boss (Solo smear frames over a bone rig), and the Sasquatch head turn, which uses bone-driven mesh points on a joystick-controlled timeline.

**Symptoms**
- Data-bound keyframe values are ignored by joystick-driven animations at runtime.
- Movement conflicts between axes: "Overlapping properties on different axes will cause movement conflicts" (Rive 101, lesson 5.5).
- An old bug kept a joystick from advancing when it was the sole component. Fixed Nov 2024 (Rive 101).

**Who:** designers, plus developers who drive head turns from cursor or app data.

**Fix**
- **Data-bound keyframe bug:** fixed in the core runtime. rive-ios notes say "Fixed data-bound keyframe values being ignored by joystick-driven, nested, and standalone animations"; rive-android lists the same fix. Update the runtime.
- **Axis conflicts:** have each axis's timeline key different properties. Head and eye X go on one axis, Y on the other.
- **App-driven poses** (community pattern, "Data Binding For Character Rigging"): don't move the joystick handle. Instead:
  1. Bind view model numbers.
  2. Combine them with a **Formula** converter and a **Range Map**.
  3. Feed the result into a **1D blend state** between two timelines, e.g. "Tilt Pos" and "Tilt Neg".
- **Cursor following:** constrain a control bone to a rectangle so the head turn never exceeds its limits. **[unofficial]**: from a DEV Community write-up.

## R6. Rendering differs between editor and runtimes

**Fixed in the Oct 1, 2026 runtime roundup**
- "Clockwise fills on bone-bound paths: winding now comes from the deformed shape, not the design-time flag."
- "Skinned fills no longer vanish after loading folded."
- "Hidden paints re-run their effects when shown." A skinned shape stroked with a trim path could previously fade in with the trim empty.

**Other differences**
- **rive-rs (Rust, Vello):** the README lists known mesh issues: "small inconsistencies at triangle borders, there can be gaps between triangles, transparent meshes will overdraw at triangle borders."
- **Editor 0.9.517** (Nov 3, 2026):
  - With Bitmap Cache off, a Component with a mesh or N-Slice draws at 1x instead of the cache resolution.
  - Clipping of artboard paints and Bitmap Cache content was fixed on WebGL2.
- **Editor 0.9.460:** small curved bone-bound paths keep the correct fill direction "when the bones mirror".
- **rive-android 11.7.0 regression** (issue #459, Jun 2026): some files rendered blank that worked on 11.6.1.

**Who:** developers.

**Fix**
- Update to runtime builds from Oct 2026 or later.
- Test rigs early on the target renderer, especially the canvas vs WebGL2 web packages and the Rust/Bevy runtimes.
- On Android, pin a known-good version when a release regresses.

## R7. Runtime performance of bones, meshes and constraints

**Official guidance** (Performance docs) is qualitative only: "Be efficient with the number of vertices in your vector image. While a few extra vertices won't have much of an impact, thousands might."

**[unofficial]** heuristic (DEV Community, "uianimation" account): "Aim for < 500 vertices for simple UI elements and < 5,000 for complex main characters." Present this as a rule of thumb, not Rive guidance.

**Measured improvements in 2026**
- **Oct 1, 2026 roundup:**
  - "Fades stop re-skinning": a 28-skin character's fade went from 100.8 to 84.8 µs/frame.
  - Rigid moves keep their path geometry: 20.8 → 15.1 µs/frame.
  - Blend states set each property once: a dashboard idle frame went from 42.9 → 14.7 µs.
  - "Cheaper updates for rigs with many hidden shapes."
  - New `drawImageMeshInstanced` for instanced image mesh drawing.
- **Editor 0.9.415 [mirror]:** Solo switches no longer re-lay out the artboard unless an option holds a layout. Large rigs with no layouts advance about 35% faster.
- **Editor 0.9.479** (Nov 1, 2026): on a rig with ~4,700 Formula converters, a frame went from 0.18 ms → 0.035 ms.

**Practical guidance**
- Put vertices only at bends.
- Cap Influence (bones per vertex) in Auto-Weights.
- Hide unused Solo options instead of fading them.
- Profile on the slowest target device.

## R8. Solos, draw order and clipping during rigging

**Context:** swapping hands, mouths or smear frames with Solos over a bone rig is standard practice.

**Fixed in Editor 0.9.642**
- Inactive Solo options showed an "old or authored size".
- The Solo's box grew to fit every option.
- In a Solo of Components, "switching to a page other than the first showed a blank page".

**Fix**
- Update to 0.9.642 or later.
- Keep draw-order rules local to the rig. Draw-order dependencies that cross a component boundary block Create Component (R4).

## R9. Gradients don't follow deformation

**Thread:** "Solved: Binding Gradients to Bones for animation". The gradient stayed fixed while a bone-bound tail bent. The staff answer pointed to binding the gradient's own control points, which confused the user. **[unclear]**: the thread is marked solved, but the exact final steps weren't visible.

**Workarounds**
- Keep gradients short and local to each deforming segment.
- Or use a raster texture on a mesh, which deforms with its triangles.

## R10. Migrating rigs from Spine, AE, Animate, Moho, Live2D

**No direct rig import** from Spine, Moho, Live2D or the After Effects rigging tools (Duik, RubberHose). Rive's own page: "There's no Spine importer. You rebuild the rig in Rive from your original art."

**Import paths that do exist**
- **Lottie / After Effects import:** Enterprise plans only. Rive's advice: "Import only to bootstrap assets, then rebuild rigging and motion inside Rive."
- **.fla import (Flash/Animate),** since Oct 2026. Animate concepts map as follows:
  - bone IK becomes opt-in IK constraints per bone;
  - Asset Warp becomes mesh deformation (raster) or vertex binding (vector).

**Workarounds**
- Rebuild the skeleton from layered PSD or vector source.
- Replace Spine physics with scripted springs.
- Replace Spine Sliders with Rive Joysticks; a one-axis joystick works as a slider.
- A coding agent with the **Rive CLI** can draft a rig, but Rive says to "check bones, weights, and IK in the Rive Editor." The CLI came out of technical preview in Nov 2026 (CLI 1.9.1).

**Note:** the Rive vs Spine page is written by Rive. Use it for stated facts (no importer, no physics constraint), not for judgments about which tool is better.

## R11. Moving a part cut from one flat image (ears, tails, hair tufts)

**Symptoms**
- A part cut out of a single flattened raster (an ear, a tail, a tuft) looks wrong when it turns: a gap or dark wedge opens at the joint, a sliver of the body moves with the part, nearby markings (stripes, spots) shear, or a spike or notch appears at the outline.
- Each fix to one end of the joint breaks the other end.

**Who:** designers and developers animating flat character art, in Rive or in plain web layers (CSS/SVG/canvas). The geometry is the same in both.

**Cause:** the part joins the body along a long root line, not at a point. A rigid rotation about any pivot moves every point on that root except the pivot, so one end of the joint lifts off and the other digs in. The wider the root and the bigger the angle, the bigger the gap (displacement ≈ distance from pivot × angle in radians; 50 px at 8° is about 7 px). Cutting the part out also leaves either a hole or leftover part pixels on the body layer.

**What does not work** (each was tried and failed on real art):
- Rigid parenting of the cut-out to a bone, or a CSS `rotate()` on the cut-out layer, with the root cut through the body.
- Putting the part *behind* the body and painting a closed body outline over the hole. The painted outline never matches the art exactly. Leftover root pixels stay on the body layer and stop moving, so they show up as notches. Changing the root cut moves the artefact somewhere else.
- Chasing each artefact with another mask tweak. If two rounds of tweaks haven't converged, the method is wrong; change the method.

**Fix: keep the root fixed and bend the part from it**
1. Draw a straight base line across the part's root. It must separate the part (above the line) from everything that must stay still (below the line). Check it against the pixels, not by eye: no body marking may sit above the line, and as much of the part as possible should sit above it. Map the region as a coarse pixel grid if needed.
2. Split the art at that line. The body layer keeps everything below it; nothing needs repainting. The part layer takes everything above it, plus a soft 3–4 px overlap below the line. A hard 1 px overlap leaves a visible hairline in browsers.
3. Keep the part layer **on top of** the body.
4. Animate a **shear along the base line**, not a rotation. Points on the line stay exactly where they are, and the tip swings sideways in proportion to its distance from the line.
   - **Web:** put `transform-origin` on the line and use `transform: rotate(A) skewX(s) rotate(-A)`, where A is the line's angle. Keep the same three functions in every keyframe so the browser interpolates `s` alone; a `matrix()` keyframe gets decomposed and stops being a pure shear.
   - **Rive:** use a mesh on the image instead of a cut-out. Bind the root-line vertices 100% to the body or head bone and the tip vertices to the part's bone, and add a ring of blended vertices in between (R1, R2). This is the Rive equivalent of the shear: the root is pinned, the tip follows the bone.
5. Keep angles small (about 8° or less) for broad parts, and check the sign of `s` for each side separately: an outward flick has opposite signs on mirrored parts.

**Verification, before saying it's fixed**
- Look at the place the user sees it: the deployed page or the real runtime, not a mock-up or a separate test render.
- Freeze the **real** animation at its extreme keyframe (in CSS, a negative `animation-delay` with `animation-play-state: paused`), zoom 3× or more, on both a light and a dark background, and inspect every joint. Also check the opposite extreme (the inward perk) and a stress angle a bit past the widest keyframe.
- Compare against the version that is live now. If the new one isn't clearly better at the same zoom, it isn't fixed.

---

## Rigging checklist

1. Convert shapes to custom paths, and add meshes to images, **before** binding. Check that View Options shows Bones.
2. Put mesh vertices only where the art bends. Run Auto-Weights with a low Influence, then Smooth at the joints. Lock finished weights.
3. Save a version before any large weight edit.
4. Put IK on the last bone and set Bone Count explicitly. Key Invert Direction or Strength for direction switches. Retest on a runtime from Oct 2026 or later.
5. Order constraints deliberately. With two at 100%, the lower one wins.
6. Rig inside the artboard that will become the component, and keep draw-order and constraint targets inside it. Use Create Component's Keep on Instance and Offset Keys (Editor 0.9.642+).
7. Give each joystick axis its own set of properties. For app-driven poses, consider view model numbers feeding a blend state.
8. Test deformation on the exact runtime and renderer you ship, and profile on the slowest device.
9. Plan to rebuild rigs from Spine, AE or Live2D rather than import them. Plan scripted physics if you need secondary motion.
10. For a part cut from one flat image, pin its root and bend from it (mesh in Rive, shear on the web). Don't rotate a cut-out with a long root (R11).
