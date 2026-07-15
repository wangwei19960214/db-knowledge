# Agent 任务启动卡片

## 每次任务只做这几步

1. 读取 `AGENTS.md`
2. 读取 `docs/INDEX.md`
3. 判断任务类型并读取对应 skill
4. 涉及问数、SQL、FineBI、报表、指标、排查或沉淀时读取 `docs/CLOSURE_PROTOCOL.md`
5. 根据任务关键词搜索相关文档
6. 只读取相关章节，不要全文读取所有文档
7. 执行任务
8. 按 `docs/CLOSURE_PROTOCOL.md` 判断是否更新文档
9. 输出对应 Closure Packet

## 不允许

- 不允许一开始全文读取所有 business 文档
- 不允许一开始全文读取所有 FineBI 文档
- 不允许为了“全面了解”学习全库
- 不允许把大段历史文档复制进上下文
- 不允许把未确认内容写成事实
- 不允许绕过入口 skill 或 `docs/CLOSURE_PROTOCOL.md` 自建一套闭环规则

## 允许

- 先搜索关键词
- 只读相关章节
- 信息不足时问用户
- 信息不影响第一版时继续推进并标注“待核实”
- 任务完成后按相关性补充已有文档
