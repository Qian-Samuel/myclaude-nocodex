# Fork 定制化修改记录

本文档记录了相对于上游 `cexll/myclaude` 的所有定制化修改，用于未来合并上游变更时参考。

## 修改摘要

| 类别 | 修改内容 | 风险等级 | 文件 |
|-----|---------|---------|------|
| 后端配置 | 默认后端从 codex 改为 claude | 低 | `agent_config.go`, `main.go` |
| 模型配置 | 各代理使用 claude 模型族 | 低 | `agent_config.go` |
| 工作流 | BMAD 子代理委派协议 | 无 | `bmad-pilot.md` |
| 安装 | BMAD 默认启用 | 无 | `config.json` |

---

## 设计原则：集中式路由

**重要**：所有后端路由决策应集中在 `agent_config.go`（或用户配置 `~/.codeagent/models.json`），工作流文档中**不应硬编码** `--backend` 参数。

正确用法：
```bash
codeagent-wrapper --agent frontend-ui-ux-engineer  # 让 wrapper 自动选择后端
```

错误用法：
```bash
codeagent-wrapper --backend gemini --agent frontend-ui-ux-engineer  # 绕过路由逻辑
```

这样做的好处：
- 修改路由规则只需改一处（agent_config.go）
- 用户可通过 `~/.codeagent/models.json` 覆盖默认配置
- 工作流文档保持稳定，不随后端策略变化

---

## 1. codeagent-wrapper 后端配置修改

### 1.1 默认后端名称 (main.go:24)

```go
// 原始值
defaultBackendName = "codex"

// 修改后
defaultBackendName = "claude"
```

### 1.2 代理模型配置 (agent_config.go:25-36)

```go
var defaultModelsConfig = ModelsConfig{
    DefaultBackend: "claude",                    // 原: "codex"
    DefaultModel:   "claude-opus-4-5-20251101",  // 原: ""
    Agents: map[string]AgentModelConfig{
        "oracle":    {Backend: "claude", Model: "claude-opus-4-5-20251101", ...},
        "librarian": {Backend: "claude", Model: "claude-sonnet-4-5-20250929", ...},
        "explore":   {Backend: "claude", Model: "claude-haiku-4-5-20251001", ...},  // 原: opencode/grok-code
        "develop":   {Backend: "claude", Model: "claude-opus-4-5-20251101", ...},   // 原: codex
        "frontend-ui-ux-engineer": {Backend: "gemini", ...},  // 保持不变
        "document-writer":         {Backend: "gemini", ...},  // 保持不变
    },
}
```

### 迁移到新架构时

上游重构后，对应文件位置变化：
- `agent_config.go` → `internal/config/agent.go`
- `main.go` → `internal/app/cli.go` 或 `internal/app/app.go`

需要在新位置做相同修改。

---

## 2. 工作流文档修改

### 2.1 dev.md (dev-workflow/commands/dev.md)

Step 0 和 Step 4 的后端路由规则更新：
- 默认后端从 `codex` 改为 `claude`
- 添加 `claude-opus` 和 `claude-haiku` 选项说明

### 2.2 SKILL.md (skills/codeagent/SKILL.md)

后端表格更新，claude 标注为默认。

### 2.3 dev-plan-generator.md

路由说明从 codex 改为 claude。

---

## 3. BMAD 工作流增强

### 3.1 bmad-pilot.md 子代理委派协议

新增 `## CRITICAL: Subagent Delegation Protocol` 部分，强制通过 Task tool 委派代理工作。

**修改位置**: 第 21-45 行, 第 337-375 行, 第 408-465 行

**目的**: 防止代理指令被内联执行，确保子代理在独立上下文中运行。

### 3.2 bmad-orchestrator.md 文件路径

新增 `## File Locations` 部分，指明全局安装路径。

### 3.3 config.json BMAD 默认启用

```json
"bmad": {
    "enabled": true,  // 原: false
    ...
}
```

---

## 4. 上游合并风险评估

### 低风险
- 后端配置修改：只涉及配置值，不涉及逻辑
- 新架构中配置文件位置清晰 (`internal/config/`)

### 中等风险
- 上游可能引入新的代理或修改代理结构
- 需要检查新架构中 `AgentModelConfig` 结构是否变化

### 需要验证
1. 新架构是否保留 `--backend` 命令行参数
2. 新架构是否保留 `~/.codeagent/models.json` 用户配置覆盖
3. Gemini 后端是否仍然可用

---

## 5. 合并步骤建议

```bash
# 1. 创建工作分支
git checkout -b merge-upstream

# 2. 合并上游
git merge upstream/master

# 3. 解决冲突后，重新应用配置修改
# 参考本文档第 1 节

# 4. 编译测试
cd codeagent-wrapper && go build && go test ./...

# 5. 验证功能
codeagent-wrapper --version
codeagent-wrapper --backend claude "test"
codeagent-wrapper --agent explore "find main function"
```

---

## 6. 相关提交历史

| Commit | 描述 |
|--------|------|
| `ceed18e` | 主要后端变更: codex → claude |
| `d5fcbf5` | 文档更新: skill 和 agent 文档 |
| `f6ec8cf` | BMAD + OmO 混合路由集成 |
| `0a58ee5` | bmad-orchestrator 路径修复 |
| `f583c94` | bmad-pilot 子代理委派协议 |

---

*文档版本*: 1.0
*创建日期*: 2026-01-22
*基于上游*: `cexll/myclaude` @ commit `90c630e` (fork 基点)
