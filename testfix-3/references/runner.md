# agent3.py 工具与产物

本页仅适用于配套项目中显式运行 `agent3.py` 的只读调查模式。Hermes 用户要求修复失败测试时，直接使用宿主的文件编辑和终端工具执行 SKILL.md 的修复流程，不调用此适配器，也不将这里的 `fixes: []` 限制用于修复模式。导入本 skill 不会自动注册下面这些运行器工具。

运行器把本 Skill 全文直接作为 agent 指令，提供可重复使用的调查工具，不向模型提供历史答案。环境继续使用项目已有的 `DEEPSEEK_API_KEY`、`DEEPSEEK_BASE_URL`、`DEEPSEEK_MODEL`；凭据不进入测试或探针子进程。

- `source_inventory()`：当前 AST 接口和源码文件清单；不是自动缺陷结论。
- `read_source(path, start_line=1, end_line=0)`：读取当前源码、测试、配置或说明，带行号；默认一次至多 400 行。大文件分页，实际已读行范围进入审查记录。
- `search_source(text)`：在当前文本文件中定位符号或契约，返回文件及行号。
- `run_tests()`：完整基线；保存 pytest.log/baseline.xml，返回真实失败和堆栈。
- `run_experiment(interface_id, hypothesis, oracle, code)`：`oracle={source,quote,reason}`，source 是当前仓库相对路径，quote 为真实文档或测试预期。code 导入真实目标，用 `check(lambda: actual, expected)` 或 `check_raises(Error, lambda: actual)`；可连续检查失败输入和正常对照，不自行定义这两个函数。从真实目标入口记录调用，失败不会阻止后续比较。失败比较与探针自身错误分别保存。普通文件、子进程、网络操作受只读审计钩子限制；它不是针对恶意原生代码的 OS 沙箱。
- `record_review(path, analysis)`：记录该源码文件的契约、已审路径及未解决问题；不能代替实际读取。
- `progress()`：尚未读/审文件、实验结果及实际剩余时间。
- `finish(findings, dispositions, partial_reason="")`：提交调查结论。每项 finding 为 `{interface_id,line,root_cause,contract_reason,disconfirmation,evidence:[experiment_id],failing_tests:[nodeid]}`。根因位置必须来自实际读过的源码；evidence 至少包含目标接口真实失败及通过对照。不同根因不能共享同一个失败实验。没有报告为缺陷的失败/错误实验须在 dispositions 中逐项解释：`{experiment_id,classification,reason}`，classification 为 `probe_error/contract_gap/blocked`。不得以新的普通通过结果抹去先前失败。

状态检查须将“调用目标”和“读取调用后的状态”放在同一个被 check 调用的辅助函数中，不能只在 setup 调用目标后比较常量。根因证据中的失败实验须追踪该根因所在的接口；需要通过上层入口触发时，interface_id 仍指定真正出错的助手。

默认完整结果要求所有源码文件读完并记录审查、每个包含函数实现的模块有实际调用目标的契约实验、基线失败已关联、实验失败均已处置。progress 会列出缺少行为实验的模块。一个模块有实验不等于覆盖其全部分支；调查笔记仍需解释选择的假设与剩余范围。部分结果必须明确 partial_reason。运行器验证的是证据与范围的一致性，语义判断仍由模型负责。

报告写在被审仓库之外：`inventory.json`、`reviews.json`、`experiments/`、`findings.json`、`fix_report.json`、`review_status.json`、`integrity.json`。fix_report 保持 `{fixes:[],unresolved:[{failing_tests,attempted}]}`；结构化调查和竞争解释保留在 findings.json。所有失败实验及更正尝试永久保留。API/工具预算结束而未成功 finish 时生成 partial 状态并失败退出，不伪造已完成报告。

入口示例：`python agent3.py --repo PATH --python PYTHON`。`--check` 仅验证 AST/工具接线，不运行目标代码或请求 API；`--effort low` 可用于小型冒烟实验，默认 high。默认 40 分钟，开始新工具操作前检查预算，不能硬中断已经发出的模型请求。
