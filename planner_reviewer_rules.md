\# Planner–Reviewer 协作协议（单文件状态机版）



\## 0. 启动规则（非常重要）



系统必须由 planner 首先启动。



初始文件必须满足：



\- role\_turn: planner

\- last\_actor: none

\- lock: free



reviewer 在任何情况下不得先于 planner 执行。



\---



\## 1. 系统概述



系统包含两个 agent：



\- planner：负责创建与修改计划

\- reviewer：负责严格审查计划



约束：



\- 只能通过【一个共享文件（TARGET\_FILE）】通信

\- 只能修改该文件

\- 必须严格遵守本协议



\---



\## 2. 文件结构（最低要求）



TARGET\_FILE 必须包含 META 区块：



role\_turn: planner | reviewer | done  

status: in\_progress | needs\_revision | approved | done | forced\_stop  

iteration: 整数  

max\_iterations: 整数  

last\_actor: planner | reviewer | none  

lock: free | planner | reviewer  



\---



\## 3. 并发控制（锁机制）



\### 3.1 写入前检查



如果：



\- lock != free 且 lock != 自己



→ 必须立即停止，不得写入



\---



\### 3.2 获取锁



写入前必须先设置：



lock: 自己



\---



\### 3.3 写入完成（顺序严格要求）



必须按以下顺序更新：



1\. last\_actor: 自己

2\. lock: free



⚠️ 不允许反序执行



\---



\### 3.4 禁止行为



\- 不允许跳过 lock

\- 不允许覆盖他人写入

\- 不允许长时间持有 lock



\---



\## 4. 回合控制（Turn-taking）



\### planner 执行条件



必须同时满足：



\- role\_turn == planner

\- status != done

\- (lock == free 或 lock == planner)

\- last\_actor != planner



否则必须立即停止



\---



\### reviewer 执行条件



必须同时满足：



\- role\_turn == reviewer

\- status != done

\- (lock == free 或 lock == reviewer)

\- last\_actor != reviewer



否则必须立即停止



\---



\## 5. Planner 行为规范



\### 5.1 初始化责任（第一次执行）



如果文件中不存在 PLAN：



planner 必须：



\- 创建完整 PLAN 结构

\- 初始化 iteration = 1

\- 设置 status = in\_progress



PLAN 结构不做强约束，但必须：



\- 可执行

\- 结构清晰

\- 包含实验/任务关键要素



\---



\### 5.2 修改阶段（status == needs\_revision）



必须：



1\. 逐条处理 reviewer 的问题（通常为 \[R#]）

2\. 不允许忽略任何 blocking issue

3\. 修改 PLAN 内容

4\. 在 PLAN 中增加：



\## reviewer\_response

\- \[R1] 修改说明

\- \[R2] 修改说明



\---



\### 5.3 reviewer 已批准（status == approved）



planner 必须判断：



\- 如果无需进一步优化：



status: done  

role\_turn: done  



\- 否则继续迭代



\---



\### 5.4 执行后更新



role\_turn: reviewer  

iteration: iteration + 1  



并清理或重置 REVIEW（如 status → pending）



\---



\## 6. Reviewer 行为规范（严格模式）



\### 6.1 核心原则



\- 默认拒绝（needs\_revision）

\- 只有在没有明显改进空间时才允许 approved



\---



\### 6.2 审查要求



必须检查：



\- 是否具体（无模糊描述）

\- 是否可执行

\- 是否关键要素缺失

\- 是否指标合理

\- 是否存在逻辑问题



\---



\### 6.3 输出格式



\#### 有问题（默认）



status: needs\_revision  



必须提供：



\## blocking\_issues

\- \[R1] 问题 + 修改建议

\- \[R2] 问题 + 修改建议



\---



\#### 无问题（极少情况）



status: approved  



\---



\### 6.4 更新 META



role\_turn: planner  



\---



\## 7. 迭代与终止机制



\### 7.1 最大轮数



如果：



iteration > max\_iterations  



任一方必须写：



status: forced\_stop  

role\_turn: done  



\---



\### 7.2 正常结束



当：



\- reviewer 给出 approved

\- planner 不再修改



→



status: done  

role\_turn: done  



\---



\## 8. 防死循环机制



\### 8.1 reviewer 不得重复问题



若问题未解决，应写：



\[R1] 未解决：原因...



\---



\### 8.2 planner 必须逐条回应



必须提供 reviewer\_response



\---



\### 8.3 空转检测



如果出现：



\- planner 未做实质修改

\- reviewer 重复反馈



reviewer 必须选择：



status: approved  



或  



status: forced\_stop  



\---



\## 9. 权限约束



\- planner 不得修改 REVIEW 内容

\- reviewer 不得修改 PLAN 内容

\- 双方仅可有限修改 META



\---



\## 10. 停止条件



满足任一：



\- status == done  

\- role\_turn == done  



→ 所有 agent 必须停止



\---



\## 11. 执行方式（推荐）



采用轮询：



循环：



1\. 读取 TARGET\_FILE

2\. 判断是否轮到自己

3\. 若否则等待

4\. 若是则执行一次



\---



\## 12. 本协议本质



这是一个：



“基于单文件的状态机协作系统”



所有行为必须由状态驱动，不允许自由发挥或绕过协议。

