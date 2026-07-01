# PPT Agent Workflow Rules

This note summarizes the rules that were most useful when building and revising `SHAP-AAD_ISCAS2026_v3.pptx`. A new agent should read this before editing the slides.

## 1. Protect User Edits First

- Always assume the user may have manually changed the PPT after the last agent edit.
- Before modifying a slide, inspect the current PPT, not an older script output.
- Only edit the requested slide range.
- If the user says a slide was manually changed and should not be touched, do not rebuild it.
- Use separate scripts for separate slide groups, for example one script for slide 6 and another for slides 7-9.
- Do not run old broad build scripts that rewrite earlier slides.

## 2. Keep Slide Text as Keywords

Slides should not contain full explanations. Details belong in the speech script.

Good slide text:

- short noun phrases;
- numbers that support the point;
- method keywords;
- result keywords.

Avoid:

- full sentences;
- long captions;
- paragraphs;
- explanatory definitions that should be spoken.

Example:

```text
Reference maps
Prediction gap
DeepLIFT approximation
Pixel SHAP -> channel ranking
```

The speech script can explain what these mean.

## 3. Use a Consistent Layout Grammar

Use one of these layouts:

- text on top, figure/table below;
- text on left, figure/table on right;
- one main figure only, if the slide is a workflow slide.

Avoid mixing many unrelated text blocks, callouts, and figures on the same page.

For result slides:

- keep the figure/table as the visual center;
- use 2-3 keyword bullets only;
- align figures in the same row by top edge and height;
- align captions by baseline;
- align table columns and keep row heights consistent.

## 4. Typography Rules

- Match the template font style, usually Times New Roman for main slide content.
- Main bullets should usually be around `26-28 pt`.
- Sub-bullets should usually be around `20-22 pt`.
- Captions should be at least `20 pt`.
- Table text should be at least `20 pt` whenever possible.
- Never allow text to overlap with figures or template page numbers.
- Do not rely only on PPT coordinates; always export screenshots because PowerPoint rendering may differ from python-pptx estimates.

## 5. Figure Rules

- Use figures from the paper when possible.
- Crop excessive white margins before inserting into PPT.
- Preserve the scientific meaning of the figure. Do not redraw a paper figure in a way that changes the representation.
- If the original method uses `32 x 32` square topographic images, show square heatmaps, not circular scalp maps.
- For workflow diagrams, redraw only when the paper figure is missing, too low-resolution, or inconsistent with the final workflow.
- Put generated images and scripts in `presentation/imgs/`.

## 6. Table Rules

- Rebuild tables directly in PowerPoint instead of inserting a screenshot if the table must be readable.
- Header row: dark blue background, white bold text.
- Important row, such as `32 channels`, can be lightly highlighted.
- Keep all table columns aligned and similar in width unless the content requires otherwise.
- Table text should be large enough for projection; target `20 pt` or above.

## 7. Caption Rules

- Captions should be short and factual.
- Avoid long figure captions copied from the paper.
- Use captions only when they help identify the figure.
- If the user removed a caption, do not add it back.

Good captions:

```text
Mean pixel SHAP map
Channel-wise SHAP ranking
Subject-wise AAD accuracy
Average accuracy
```

## 8. Visual Balance

- Do not leave large unused blank regions unless the slide intentionally needs breathing room.
- Do not fill space with decorative shapes.
- Avoid heavy callout boxes unless they summarize one essential takeaway.
- Keep left/right margins balanced.
- If two figures are in one row, their top edges should align.
- If two figures are in one column, their widths should match.

## 9. Scientific Accuracy Rules

Do not simplify a method beyond what is technically correct.

For DeepSHAP in SHAP-AAD:

- the CNN predicts binary auditory attention direction from `32 x 32` alpha-power maps;
- DeepSHAP is post-hoc and does not train the CNN;
- Shapley value is the theoretical contribution target;
- exact Shapley computation is infeasible for 1024 image pixels;
- DeepLIFT provides efficient difference-from-reference backpropagation;
- DeepSHAP combines SHAP-style attribution with DeepLIFT-style propagation and multiple background references;
- pixel attributions are averaged and mapped back to electrodes for channel ranking.

## 10. Workflow for Each Editing Request

1. Inspect the current PPT file and the target slide shapes.
2. Check nearby slides for font size, title position, and visual style.
3. Identify the slide's one message.
4. Decide the layout: top-text/bottom-figure or left-text/right-figure.
5. Prepare/crop images in `presentation/imgs/`.
6. Write a narrow update script for only the requested slides.
7. Run the script.
8. Export screenshots with PowerPoint COM.
9. Visually inspect screenshots for overlap, alignment, small text, and excessive blank space.
10. Iterate until the screenshot looks correct.
11. Run a structure check for small explicit text and accidental manual page numbers.

## 11. Useful Verification Checks

Check visible text size and accidental manual page numbers:

```bash
python - <<'PY'
from pptx import Presentation
import re
prs = Presentation('presentation/SHAP-AAD_ISCAS2026_v3.pptx')
for si in range(1, len(prs.slides)+1):
    small = []
    slide = prs.slides[si-1]
    for i, sh in enumerate(slide.shapes):
        if getattr(sh, 'has_text_frame', False):
            for p in sh.text_frame.paragraphs:
                for r in p.runs:
                    if r.text.strip() and r.font.size is not None and r.font.size.pt < 20:
                        small.append((i, r.font.size.pt, r.text))
        if sh.shape_type == 19:
            for ri, row in enumerate(sh.table.rows):
                for ci, cell in enumerate(row.cells):
                    for p in cell.text_frame.paragraphs:
                        for r in p.runs:
                            if r.text.strip() and r.font.size is not None and r.font.size.pt < 20:
                                small.append((f'table {ri},{ci}', r.font.size.pt, r.text))
    if small:
        print('slide', si, small)

manual = []
for si, slide in enumerate(prs.slides, start=1):
    for i, sh in enumerate(slide.shapes):
        if getattr(sh, 'has_text_frame', False) and re.fullmatch(r'\d+\s*/\s*\d+', sh.text.strip()):
            manual.append((si, i, sh.text.strip()))
print('manual page numbers:', manual)
PY
```

Export a slide screenshot:

```bash
powershell.exe -ExecutionPolicy Bypass -File 'E:\wps\Projects\ISCAS2026\paper_material\presentation\imgs\export_ppt_slide.ps1' -PptxPath 'E:\wps\Projects\ISCAS2026\paper_material\presentation\SHAP-AAD_ISCAS2026_v3.pptx' -OutputPath 'E:\wps\Projects\ISCAS2026\paper_material\presentation\imgs\slide_check.png' -SlideNumber 7 -Width 1920 -Height 1080
```

## 12. Main Lesson

The most reliable process is:

```text
read current PPT -> make a narrow script -> export screenshot -> compare visually -> iterate
```

Good PPT work here was not mainly about adding more design elements. It improved when the slides became simpler, more aligned, and more faithful to the paper workflow.

