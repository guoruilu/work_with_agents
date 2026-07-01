# PPT 制作与 reviewer-executor 复审经验

最后更新：2026-05-16

本文件记录本次 `SHAP-AAD_Slides.pptx` 制作、严格复审、执行修复到收敛的经验。目标是给后续 agent 做论文介绍 PPT 或 poster 时直接参考。

## 1. 必须闭环，而不是只改一轮

- reviewer 和 executor 要分工明确：reviewer 只审查并输出可执行问题清单，executor 只按清单修改并重建产物。
- 每轮修改后必须重新导出截图，再交回 reviewer 复审。
- 停止条件必须同时满足：
  - reviewer 明确给出 `CONVERGED`；
  - executor 明确确认 `AGREES_CONVERGED`，没有未完成或仍值得做的修改。
- 日志里曾经写过“收敛”不能作为证据；只能以当前脚本、当前 PPTX、当前截图和论文原文为准。

## 2. 截图是硬证据

- `python-pptx` 的几何坐标不等于 PowerPoint 最终渲染结果。字体替换、行距、符号宽度都会导致溢出。
- 每次改完必须运行：

```bash
python build_shap_ppt.py
powershell.exe -ExecutionPolicy Bypass -File screenshot.ps1
```

- 检查截图时优先看：
  - textbox 是否有文字溢出到边框外；
  - 图、表、caption 是否互相遮挡；
  - 底部 take-away 条是否压住正文；
  - 小图是否只是装饰，还是能承担叙事功能；
  - 箭头是否穿过错误模块或表达了错误的数据流。

## 3. 事实表述要区分“论文声称”和“讲者推断”

- PPT 为了讲清楚，可以加入 interpretation，但必须标清。
- 对本项目尤其要严格区分：
  - Paper claim：TCN 直接处理 raw EEG，可以保留 temporal dynamics；降通道后 topomap 插值可能引入 spatial artifacts。
  - Additional interpretation：直接对 TCN 做 SHAP 可能把 channel attribution 和 temporal filters 纠缠在一起。
- 不要把推断写成论文结论，例如 “SHAP on TCN is circular” 这种强断言，除非论文原文明确这样说。

## 4. 百分比和百分点不能混用

- 结果页要写清楚基线：
  - `64 -> 48: -1.79 pp`
  - `48 -> 32: additional -0.06 pp`
  - `64 -> 32 total: -1.85 pp`
- `0.06 pp` 只表示 32 通道相对 48 通道几乎不降，不能写成 64 到 32 只降 0.06。
- 81.06% 到 79.21% 是 1.85 percentage points，不是 1.85% relative accuracy。

## 5. 数学页宁可少写，也不要像代码字符串

- `Σ_{S ⊆ N\\{i}}`、`m_{x₁→h}` 这类 raw notation 在 PPT 截图里容易像源码，不像公式。
- 如果没有可靠的公式渲染流程，优先改成教学表达：
  - “weighted average over coalitions S”
  - “m(x1 -> h)”
  - “exact weight for coalition S”
- 复杂公式可以放在讲稿或补充文档里，PPT 只保留足以支撑理解的核心关系。

## 6. 机制图的箭头必须表达真实计算流

- DeepSHAP schematic 不能画成 `f(x)-E[f(x')]` 直接生成 `SHAP values`。
- 正确视觉语义应是：
  - forward：`x` 和 `x'` 通过 trained CNN 得到 `f(x)` 和 `E[f(x')]`；
  - backward：输出差异经 trained CNN 反传到 input feature deltas，再形成 `φ_i`。
- 如果 caption 写 “through f”，图中也必须看得出路径经过 CNN；否则观众会按图理解成错误机制。

## 7. 图表要互相解释，不能互相打架

- 如果 Fig. 6 展示 60-channel bar，而表格没有 60 行，要显式写 “Fig. 6 includes 60-channel; table shows selected rows”。
- 标题说 “pixel map”，顶部 input/output 图就不能过小；否则视觉权重会被 ranking 图抢走。
- 结果页不要写 “matches full baseline” 这类过强营销句；论文语气通常是 “comparable”、“near-baseline” 或 “retains most accuracy”。

## 8. 源文档也要同步修正

- PPT 修正了事实表述后，Markdown 源文档里的同类风险也要修。
- 否则后续 agent 从文档重新抽文案时，会把旧错误带回 PPT。
- 本次已同步修正中英文介绍文档中关于 `64 -> 32` 和 `48 -> 32` 的歧义。

## 9. 对后续 agent 的建议流程

1. 读论文原文和已有 Markdown，不只读 LOG。
2. 先生成或更新 PPTX。
3. 导出 PNG 截图。
4. reviewer 严格审查：事实、视觉、教学、脚本维护性。
5. executor 逐条修复，不做无关重构。
6. 继续 reviewer -> executor 循环，直到双方都明确收敛。
7. 最后写日志，记录修改项和验证项。
