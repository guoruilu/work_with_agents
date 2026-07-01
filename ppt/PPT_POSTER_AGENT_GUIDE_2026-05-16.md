# PPT/Poster Agent 工作指南

## 1. Agent 做 PPT 或 Poster 必须遵守的原则

1. 图尽可能用原图；如实在要重画，也必须与原论文保持一致。
2. reviewer 和 executor 都用子 agent 来做，主 agent 只用来协调、分派任务、汇总状态和决定是否继续循环。
3. 事实优先于美观。所有数字、结论、图注、方法流程和表格说明必须能回到论文、已有文档或本地数据中验证。
4. 表述必须克制。论文只支持 comparable、suggests、highlights 时，不要写成 matches、negligible、clearly proves、dominates 或 drives。
5. 科技海报中百分比和百分点必须区分清楚。准确率差值写 pp，例如 `81.06% -> 79.21%` 应写 `-1.85 pp`，不是 `-1.85%`。
6. 图表必须服务现场阅读。远距离不可读的小字、密集 tick、过小 legend、不可辨认坐标轴和颜色含义，都应视为需要修改的问题。
7. 版式不能制造误解。截断轴柱状图、缺行表格、流程图错误箭头、过度简化图注都会让读者得到错误信息，必须优先修正。
8. 页脚、联系方式、脚注和 equal contribution 说明要能被看见；不可塞在过小、过暗或贴近边缘的位置。

## 2. 具体操作方法

1. 先备份当前 PPT/PDF，文件名包含日期、时间和来源说明，例如 `SHAP-AAD_Poster_claude-code_backup_YYYY-MM-DD_HHMM.pptx`。
2. 阅读本地材料：论文 PDF 或文本、PPT 指南、已有 notes、生成脚本、截图和当前 PPT/PDF。不要只看页面外观。
3. 让 reviewer 子 agent 进行严格审查，输出分级问题清单，至少覆盖事实、措辞、格式、图表可读性、版式、美观和观众理解风险。
4. 让 executor 子 agent 只修改 reviewer 指出的范围。修改前确认涉及文件，修改后重建 PPT/PDF 和截图。
5. 每轮都导出截图复查。不要只相信 PPT XML 或脚本参数；必须看最终渲染结果。
6. 对关键数字逐项核对：准确率、差值、参数量、MFLOPs、通道数、图号、表号和 caption。差值单位优先写 `pp`。
7. 对论文方法流程做因果核对。比如 SHAP/CNN 只用于得到 top-k channel mask 时，流程图应显示 mask 回到 time-series EEG，再训练 TCN，不能画成 CNN 输出直接进入 TCN。
8. 对缺失或抽样展示必须显式说明。比如图中有 60-channel 而表格只列 selected rows，caption 应写清 `60 ch appears only in Fig. X; ΔAcc vs. 64 ch.`。
9. 图表重画时要避免视觉误导。若使用截断 y 轴，优先改为点线图；如必须用柱状图，应从 0 起始或明确画断轴标记。
10. 图表文字按 poster 场景放大。坐标轴、legend、tick、图内标签通常应达到 24-28 pt；表格表头和内容也要满足本地指南。
11. 图注要解释颜色和语义，例如 `warm = higher alpha power`、`bright = higher |SHAP|`，不要只写 `high`。
12. reviewer-executor 循环持续到 reviewer 明确给出 `CONVERGED`，且主 agent 确认没有新的用户要求或未处理风险后再停止。

## 3. 推荐用户给予的支持

1. 提供最终提交场景：会议模板、尺寸、横竖版、打印距离、是否需要 PDF、是否必须保留学校或项目 logo。
2. 提供权威材料：论文最新版、补充材料、原始图、数据表、代码仓库、作者排序、单位、邮箱、脚注和引用格式。
3. 明确哪些内容不能改：标题、作者、配色、logo、图号、实验数字、核心 claim、会议或导师指定措辞。
4. 明确哪些内容可以重画：流程图、结果图、表格、图注、参考文献、联系信息和局部版式。
5. 对核心 claim 给出人工确认，尤其是 SOTA、novel、first、significant、negligible、matches 等高风险词。
6. 提供偏好的审稿强度。若用于正式投稿或大会展示，应默认使用最严格 reviewer，并允许多轮 executor 修改。
7. 在每轮后尽快指出主观偏好，例如颜色、拥挤程度、是否保留完整 references、是否加 QR code。
8. 保留可复现入口：最好用脚本生成 PPT/PDF，并让 agent 记录生成命令、截图命令和输出文件路径。
