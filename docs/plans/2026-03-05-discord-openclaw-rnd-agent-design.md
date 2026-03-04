# Discord + openclaw 研发自动化多 Agent 协作设计

## 1. 背景与目标

### 背景
用户希望在 Discord 中搭建基于 openclaw 的多 Agent 分工协作体系，用于研发自动化任务流。

### 目标范围（已确认）
- 流程类型：研发自动化流
- 仓库形态：单仓库（MVP）
- 执行环境：本机执行
- 自动化边界：可改本地代码并可跑测试，但不自动 commit/push
- 编排形态：openclaw + Discord Bot 编排层
- 编排技术：Python

### 成功标准
- Discord 可发起、查询、终止任务
- 任务在固定四阶段 Agent 串行执行稳定
- 产出包含改动摘要、测试结果、风险与下一步建议
- 全程可追溯（按 task_id 留痕）

---

## 2. 方案对比与结论

### 方案 A（推荐）：中心调度器 + 专职 Agent 管线
- 组件：Discord Bot、Python Orchestrator、4 个专职 Agent、Artifact 存储
- 流程：planner → coder → tester → reviewer（串行）

**优点**
- 职责边界清晰，问题定位快
- 易扩展（后续可新增 security/docs 等 Agent）
- 适合研发任务的稳定闭环

**缺点**
- 需要 upfront 定义 Agent 输入/输出契约

### 方案 B：单 Agent + 工具插件
**优点**：上线最快。
**缺点**：复杂任务下稳定性与可维护性差。

### 方案 C：事件总线并行 Agent
**优点**：吞吐与扩展性强。
**缺点**：MVP 过重，当前阶段投入不划算。

### 结论
采用**方案 A**作为首版落地架构。

---

## 3. 架构设计

### 3.1 总体组件
1. **Discord Bot**：接收 Slash Command 并回传进度/结果
2. **Orchestrator（Python）**：任务编排、状态机、重试/超时、审计
3. **Agent 层（openclaw）**：
   - planner（拆解任务与验收标准）
   - coder（修改代码）
   - tester（执行测试）
   - reviewer（汇总结论）
4. **Artifact Store（本地）**：每个 task_id 独立目录保存中间件与日志

### 3.2 核心流程
1. 用户执行 `/task <目标>`
2. Bot 生成 task_id 并入队
3. Orchestrator 串行调用四阶段 Agent
4. 结果落盘并通过 Discord 回帖
5. 全流程不触发 commit/push

### 3.3 状态机
- `created` → `planning` → `coding` → `testing` → `reviewing` → `completed`
- 任一阶段异常：进入 `reviewing` 输出失败摘要与建议，不中断可解释性

---

## 4. Agent 契约设计（JSON）

### 4.1 统一输入信封
```json
{
  "task_id": "2026-03-04-001",
  "repo_path": "/abs/path/to/repo",
  "branch": "master",
  "goal": "修复登录超时问题",
  "constraints": {
    "allow_commit": false,
    "allow_push": false,
    "max_runtime_sec": 900
  },
  "artifacts_dir": ".openclaw/artifacts/2026-03-04-001"
}
```

### 4.2 planner 输出
```json
{
  "subtasks": ["定位超时触发点", "修复重试逻辑", "补充回归测试"],
  "acceptance_criteria": ["弱网下登录成功率提升", "现有测试全绿"],
  "test_commands": ["pytest -q"]
}
```

### 4.3 coder 输出
```json
{
  "changed_files": ["src/auth.py", "tests/test_auth.py"],
  "patch_summary": ["调整超时重试次数", "补充测试用例"],
  "status": "patched"
}
```

### 4.4 tester 输出
```json
{
  "status": "failed",
  "results": [
    {"cmd": "pytest -q", "exit_code": 1, "report_file": "pytest.log"}
  ]
}
```

### 4.5 reviewer 输出
```json
{
  "final_status": "needs_attention",
  "summary": "核心逻辑已修改，但有 1 个测试失败",
  "risks": ["边界条件未覆盖"],
  "next_actions": ["修复 test_auth::test_timeout_retry"]
}
```

---

## 5. Discord 指令面

## 5.1 指令集合（MVP）
- `/task <目标>`：创建并执行任务
- `/status <task_id>`：查看阶段与进度
- `/result <task_id>`：查看最终摘要与产出
- `/abort <task_id>`：中止任务并保留现场

### 5.2 权限模型
- **Admin**：可 `/abort`、管理配置
- **Developer**：可 `/task` `/status` `/result`
- **Viewer**：可 `/status` `/result`

Discord Role 与本地权限映射配置化管理（YAML/JSON）。

---

## 6. 安全与治理

1. **仓库路径白名单**：仅允许固定 repo_path，阻断路径逃逸
2. **命令白名单**：仅允许运行测试/lint/受控脚本
3. **Git 安全闸**：allow_commit/allow_push 强制为 false
4. **资源控制**：任务级超时（建议 15 分钟）
5. **审计日志**：记录触发人、命令、退出码、改动文件、阶段耗时

---

## 7. 失败处理策略

- planner 失败：返回“需求补充项”模板
- coder 失败：保留改动快照与错误日志
- tester 失败：输出失败摘要与日志路径
- reviewer：始终产出可执行的下一步建议

---

## 8. 实施路径（MVP）

1. 创建 Discord Bot 与 Slash Commands
2. 实现 Python Orchestrator（队列 + 状态机 + 落盘）
3. 接入 openclaw 四阶段 Agent 与 schema 校验
4. 加入安全闸（路径/命令白名单、超时）
5. 回传进度与结果到 Discord
6. 用 3 类任务做验收（小 bug/小功能/小重构）

---

## 9. 建议目录结构

```text
project-root/
  bot/
    discord_bot.py
    commands.py
  orchestrator/
    runner.py
    state_machine.py
    task_store.py
  agents/
    planner.yaml
    coder.yaml
    tester.yaml
    reviewer.yaml
    schemas/
      planner_output.json
      coder_output.json
      tester_output.json
      reviewer_output.json
  security/
    command_allowlist.yaml
    repo_allowlist.yaml
  .openclaw/
    artifacts/
      <task_id>/
        planner.json
        coder.json
        test.log
        reviewer.json
        summary.md
  docs/
    plans/
```

---

## 10. MVP 完成定义（DoD）

- 可通过 Discord 发起并追踪任务
- 四阶段串行稳定可复现
- 可改代码 + 可测试 + 禁止自动提交
- 失败场景下仍能给出可执行下一步
- 按 task_id 的日志与产物可完整追溯
