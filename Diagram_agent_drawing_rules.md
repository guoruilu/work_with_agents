# Diagram Agent Drawing Rules

This document defines general rules for agents that redraw, reconstruct, or create editable technical diagrams. It complements `PPT_agent_general_workflow_rules.md`: the PPT rules focus on deck editing workflow, while this document focuses on drawing fidelity, reference matching, geometry, layering, and visual verification.

## 1. Decide the Drawing Goal First

Before drawing, classify the task:

- **Reference reconstruction**: reproduce an existing image as editable shapes.
- **Semantic redraw**: preserve meaning while simplifying or improving style.
- **New diagram**: create a diagram from text, data, or code.
- **PPT-native redraw**: convert bitmap content into editable PowerPoint objects.

The strictness of visual matching depends on this goal. If the user says "redraw this figure so I can edit it", prefer reference reconstruction with editable primitives.

## 2. Use the Reference Image as the Coordinate System

For reference reconstruction:

- Read the reference image size first.
- Match the output canvas aspect ratio to the reference.
- Use reference-pixel coordinates as the master coordinate system whenever possible.
- Convert reference pixels to PPT, SVG, canvas, or other target units through one scale factor.
- Do not guess coordinates in inches if a pixel reference exists.
- Keep generated output at the same screenshot/export resolution as the reference during checks.

Good pattern:

```text
reference image: 3926 x 1185
slide aspect: 3926 / 1185
all object coordinates: source pixels
render check: export at 3926 x 1185
```

This makes differences visible and avoids distortion from mismatched canvas ratios.

## 3. Trace Geometry Before Styling

Build the diagram in this order:

1. Canvas size and coordinate conversion.
2. Major object bounding boxes.
3. Object grouping and z-order.
4. Connectors and arrows.
5. Labels and text boxes.
6. Fine details such as dots, dashes, dividers, and line endings.
7. Colors, borders, transparency, and shadows.

Do not start by choosing colors or decorative style. Most failures come from wrong geometry, not wrong colors.

## 4. Extract Measurable Reference Features

When a reference image is available, use visual inspection plus simple measurement:

- crop relevant regions;
- measure image dimensions;
- detect approximate bounding boxes for colored blocks;
- detect dark lines, dots, or masks when useful;
- compare crop-to-crop rather than whole-image only.

Useful things to measure:

- block left/top/right/bottom;
- line start/end points;
- arrowhead size and angle;
- text baseline area;
- repeated item spacing;
- table or vector cell sizes;
- object overlap and z-order.

Do not overfit every pixel. Use measurement to correct large proportional errors.

## 5. Preserve Editable Structure

When the user wants to modify the diagram later:

- Prefer native shapes over inserted screenshots.
- Use separate shapes for logically separate components.
- Name shapes descriptively when the API allows it.
- Keep repeated components generated from data structures.
- Avoid flattening groups into bitmaps unless the part is truly decorative or too complex.
- Use reusable drawing helpers for repeated primitives.

Examples of editable primitives:

- rectangles for frames, tables, and output classes;
- freeform polygons for 3D blocks;
- connectors plus polygon arrowheads for arrows;
- text boxes for labels;
- small circles for points or event dots.

## 6. Respect Z-Order and Layering

Layering is part of the drawing, not an implementation detail.

For stacked or 3D diagrams:

- Draw back objects first and front objects later.
- Draw internal details before foreground blocks if they should be hidden behind a block.
- Draw connector lines after blocks only when they should be visible on top.
- If a dash or arrow appears to pass behind a module, draw it before that module or split it into visible segments.
- Verify the rendered output, because API order and PowerPoint rendering can differ from code assumptions.

Common failure:

```text
draw long dashed line on top of all blocks -> line cuts through modules
```

Better:

```text
draw short visible dash segments only in the gaps, or draw the line before foreground blocks
```

## 7. Match Repeated Structures by Rhythm

For repeated blocks, cells, bars, or frames, match the rhythm:

- count;
- relative size;
- spacing;
- stagger direction;
- top and bottom alignment;
- depth or slant angle;
- front/side/top face proportions;
- repeated label alignment.

A technically correct diagram can still look wrong if repeated items have the wrong rhythm. Thin repeated blocks should remain thin; grouped blocks should remain visually independent.

## 8. Draw 3D Blocks Carefully

For oblique 3D blocks:

- Represent each block as front, side, and top faces.
- Keep the slant direction consistent across the diagram.
- Use separate colors for front, side, and top faces.
- Match face proportions from the reference.
- Avoid making depth vectors too long; over-deep blocks look heavy.
- Do not let large side faces swallow adjacent thin blocks.
- Use reference bounding boxes to calibrate front-face width and depth.

When a block looks too heavy, first reduce depth and front width before changing colors.

## 9. Arrows and Dashes Need Explicit Control

Arrows are often where redrawn diagrams diverge from references.

Rules:

- Match arrow start points, end points, slope, and arrowhead size.
- Use separate independent arrows if the reference shows independent arrows.
- Do not replace independent arrows with a single fan-out node unless the reference does that.
- For dotted or dashed arrows, match line weight and dash rhythm as closely as the target tool allows.
- If built-in dash styles are too coarse, draw explicit short segments.
- Keep arrowheads from touching or covering target shapes unless the reference does.

For network or pipeline diagrams, connectors are semantic. Wrong fan-out geometry can imply a different computation graph.

## 10. Text Boxes Must Be Verified Rendered

Text that looks fine in code can wrap or clip after rendering.

Rules:

- Use text boxes large enough for the longest label.
- Disable word wrap for labels that must remain one line.
- Check rotated labels after export.
- Check baseline and vertical centering in output cells.
- Keep labels inside the canvas bounds.
- Avoid relying on estimated text metrics.

Always render the diagram and inspect actual text layout.

## 11. Synthetic Detail Should Match the Reference's Density

For dots, noise, event clouds, scatter points, or texture-like marks:

- Match the rough density, concentration, and region.
- Match dominant colors and point size.
- Use deterministic random seeds so results are reproducible.
- Do not make synthetic details too clean if the reference is noisy.
- Do not let random dots hide important labels or arrows.

If exact data is unavailable, reproduce the visual role, not every point.

## 12. Separate Semantic Accuracy From Visual Fidelity

Check both:

- **Semantic accuracy**: object count, labels, order, flow direction, categories, method meaning.
- **Visual fidelity**: proportion, spacing, alignment, line weight, color, density, z-order.

Do not accept a diagram just because the semantics are correct if the user asked for a close redraw.
Do not overfit visuals in a way that changes the scientific or technical meaning.

## 13. Use Narrow Scripts and Data-Driven Coordinates

For repeatable drawing:

- Write a task-specific script instead of manual one-off edits.
- Keep coordinates and repeated objects in data lists.
- Keep helper functions small: `add_text`, `add_line`, `add_block`, `add_rect`, etc.
- Regenerate the output from the script after every geometry change.
- Avoid stale scripts that no longer match the current drawing.

Good structure:

```text
constants -> helpers -> input area -> backbone -> head -> output -> build()
```

This makes later edits safer and makes shape placement auditable.

## 14. Compare Crops, Not Only the Whole Figure

After rendering:

- compare the full figure;
- crop high-risk regions and compare them separately;
- inspect left, center, right, and bottom label areas;
- inspect any area that was criticized in the previous iteration.

Whole-figure checks miss local errors. Crops reveal wrong spacing, hidden dots, clipped labels, and line overlap.

## 15. Use Independent Visual Review

When fidelity matters, use a separate reviewer or sub-agent after export.

Ask the reviewer to compare:

- reference image;
- current rendered output;
- specific areas changed in the latest iteration;
- whether remaining differences are blocking or acceptable.

The drawing agent should not be the only judge of its own rendered output.

Recommended review categories:

```text
Needs modification
Acceptable differences
Matched well
Accept / Do not accept
```

## 16. Iterate on Concrete Findings

Do not broadly redraw everything after each review.

Use this loop:

```text
render -> independent review -> list concrete failures -> patch only those failures -> render again
```

Examples of concrete failures:

- "output label wraps";
- "gold block too thick";
- "fan-out should be three independent arrows";
- "dots are hidden behind frame";
- "dash segments are too long and cut through modules";
- "bottom title is too low".

Avoid vague fixes like "make it more similar". Convert them into measurable geometry changes.

## 17. Acceptance Criteria

A redrawn diagram is acceptable when:

- the canvas ratio matches the target;
- all required labels are present and readable;
- object counts and flow direction are correct;
- key proportions and spacing match the reference closely enough for the stated goal;
- no important shapes are clipped, hidden, or unintentionally overlapped;
- arrows and dashes do not imply a different structure;
- text does not wrap or overflow unexpectedly;
- an exported screenshot has been inspected locally;
- an independent reviewer accepts the rendered output when fidelity matters.

## 18. Common Pitfalls

- Drawing on a 16:9 canvas when the reference has a different aspect ratio.
- Using approximate inch coordinates instead of reference-pixel coordinates.
- Making 3D block depth too large.
- Drawing dashed lines on top of modules so they cut through shapes.
- Letting text labels wrap after export.
- Moving one group without moving associated arrows and labels.
- Assuming synthetic dots are visible when they are hidden behind later shapes.
- Treating arrow fan-out geometry as decorative when it carries meaning.
- Accepting the result without a same-resolution screenshot.
- Ignoring sub-agent or reviewer feedback because the local view looks "good enough".

## 19. Minimal Agent Instruction

When giving this to another agent, this short version is enough:

```text
Redraw the reference as editable shapes. Use the reference image's pixel coordinate system and match the output canvas aspect ratio. Build geometry first, then style. Preserve editable objects and correct z-order. Export a same-resolution screenshot. Compare full image and local crops against the reference. Ask an independent sub-agent to judge visual fidelity. Iterate only on concrete differences until the reviewer accepts.
```

## 20. Core Principle

Reliable diagram drawing follows this process:

```text
reference pixels -> editable primitives -> same-size render -> independent comparison -> targeted iteration
```

The key lesson is that visual fidelity comes from measurable geometry, correct layering, and rendered verification, not from approximate manual placement.
