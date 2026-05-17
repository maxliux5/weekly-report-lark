# weekly-report-lark

从飞书（Lark/Feishu）自动收集证据、生成结构化周报的 Claude Code Skill。

> **核心哲学**：周报不是写作问题，而是上下文收集问题。不要先写 prose，先收集证据，再分类，再归纳。

---

## 功能亮点

- **自动收集证据**：从任务、日程、会议、聊天记录、近期编辑文档中自动拉取本周工作素材
- **项目维度组织**：不按时间流水账，而是按项目聚合多来源证据
- **区分所有权**：明确标注"主导/参与/待跟进"，杜绝夸大
- **基于证据**：没有聊天记录、任务或文档编辑支撑的事项，绝不编造
- **一键回写飞书**：生成后直接追加到你的周报汇总文档

---

## 前置依赖

1. [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview) 已安装并配置
2. `lark-cli` 已安装并在 PATH 中可用
3. lark-cli 已完成飞书认证，至少具备以下权限：
   ```bash
   lark-cli auth login --domain calendar,task,vc,drive,docs
   lark-cli auth login --scope "search:message search:docs:read"
   ```

---

## 安装

**零配置即可使用**，唯一要求是 `lark-cli` 已安装并完成飞书认证。

```bash
# 1. 克隆或复制本 Skill 到 Claude Code 的 skills 目录
mkdir -p ~/.claude/skills/weekly-report-lark
cp -r . ~/.claude/skills/weekly-report-lark/

# 2. （可选）如果你想自定义配置或项目映射
mkdir -p ~/.weekly-report-config
cp config.example.json ~/.weekly-report-config/config.json
# 编辑 ~/.weekly-report-config/config.json
```

---

## 快速开始

在 Claude Code 中直接说：

```
帮我写这周周报
```

Claude 会自动：
1. 检查 `lark-cli` 是否就绪
2. 读取 `~/.weekly-report-config/config.json`（如有），否则使用默认配置
3. 从飞书拉取本周证据
4. 按项目归类、区分所有权
5. 生成周报；如无目标文档，自动在飞书创建"周报汇总"文档并写入
6. 把新文档的 URL 保存到 `~/.weekly-report-config/config.json`，下次直接复用

### 其他常用指令

| 指令 | 效果 |
|------|------|
| "帮我写这周周报" | 生成本周（周一至今）的周报 |
| "整理一下上周做了什么" | 生成上一整周的周报 |
| "从飞书里拉素材生成周报" | 同上，显式触发证据收集 |
| "把周报更新到文档里" | 生成并追加到配置的目标文档 |
| "只生成聊天内容不要写回去" | chat-only 输出，不回写文档 |

---

## 配置说明

### 配置文件存放位置

**所有配置都是可选的。** 只要 `lark-cli` 能正常工作，skill 就能自动创建文档并运行。

如需自定义，配置优先从用户目录读取，方便升级 skill 时不被覆盖：

1. `~/.weekly-report-config/config.json` — 用户个人配置（优先）
2. `~/.claude/skills/weekly-report-lark/config.json` — skill 目录配置（回退）

首次安装时，从仓库模板复制：
```bash
mkdir -p ~/.weekly-report-config
cp config.example.json ~/.weekly-report-config/config.json
```

### config.json

```json
{
  "weekly_report_doc_name": "我的周报汇总",
  "weekly_report_doc_url": "https://your-org.larkoffice.com/docx/XXXXXXXX",
  "output_format": "feishu",
  "auto_write_back": true,
  "ownership_labels": {
    "lead": "我主导/负责的",
    "participate": "我参与评审/同步的",
    "follow_up": "后续需要我跟进的"
  }
}
```

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `weekly_report_doc_name` | 周报文档的名称 | `"周报汇总"` |
| `weekly_report_doc_url` | 飞书文档的完整 URL，作为写入目标 | 无配置时自动创建 |
| `output_format` | 输出格式：`feishu` / `markdown` / `plain` | `feishu` |
| `auto_write_back` | 是否自动回写到飞书文档 | `true` |
| `ownership_labels` | 三个所有权分类的自定义文案 | 见示例 |

### project-mapping.json

通过关键词自动将任务/会议/文档映射到项目名，减少 AI 推理的模糊性。

存放位置同样优先从用户目录读取：
1. `~/.weekly-report-config/project-mapping.json`
2. `~/.claude/skills/weekly-report-lark/project-mapping.json`

```json
{
  "mappings": {
    "前端": "前端架构 / 组件库建设",
    "后端": "后端服务 / API 开发",
    "AI": "AI 功能接入 / 模型对接",
    "文档": "技术文档 / 知识沉淀",
    "监控": "稳定性 / 监控告警",
    "稳定性": "稳定性 / 监控告警"
  }
}
```

当证据标题或内容包含关键词时，优先归入对应项目。未匹配的关键词由 AI 自动推断。

---

## 数据来源与优先级

Skill 按以下顺序收集证据，证据充足时自动停止：

1. **任务中心** — 已完成/进行中的任务（最干净的结构化信号）
2. **日历/日程** — 推断协调工作量、评审/同步时间
3. **会议纪要 & 待办** — 决策、后续行动、阻碍
4. **聊天记录** — 补充不在任务中的重要进展
5. **近期编辑文档** — 直接产出证据，自动生成可点击链接

---

## 输出示例

见 [`examples/weekly-report-example.md`](examples/weekly-report-example.md)。

---

## 故障排查

| 现象 | 排查步骤 |
|------|----------|
| `lark-cli: command not found` | 确认 lark-cli 已安装且在 PATH 中。尝试 `which lark-cli` 验证。 |
| `auth required` / 401 错误 | 运行 `lark-cli auth login --domain calendar,task,vc,drive,docs` 重新授权。 |
| 找不到周报文档 | 无配置时 skill 会自动创建新文档。如创建失败，检查 `lark-cli` 的 docs 权限；也可手动创建文档后把 URL 写入 `~/.weekly-report-config/config.json`。 |
| 文档搜索无结果 | 确认 auth scope 包含 `search:docs:read`。如缺失，降级为 tasks + calendar + meetings。 |
| 会议纪要为空白 | 部分会议未开启自动纪要。尝试用 calendar + chat 补充该会议相关证据。 |
| 生成的周报内容太少 | 可能是该周任务/会议较少，或 lark-cli 某些数据源返回为空。AI 会明确告知缺失了哪些来源。 |
| 项目分组不准确 | 在 `project-mapping.json` 中添加更多关键词映射，提升匹配精度。 |

---

## 自动化触发（进阶）

你可以结合系统的 cron 或 Claude Code 的定时任务，实现"每周五下午 5 点自动生成周报草稿"。

### macOS / Linux (cron)

```bash
# 每周五 17:00 自动触发
crontab -e
0 17 * * 5 cd /path/to/your/project && claude --skill weekly-report-lark "帮我写这周周报"
```

### Claude Code Scheduled Tasks

如果你使用 Claude Code Desktop 或支持 scheduled tasks 的环境：

```
/schedule "每周五 17:00 自动生成周报草稿" /skill weekly-report-lark 帮我写这周周报
```

> 注意：cron 方式运行时需确保 `lark-cli` 的 auth token 在运行环境下有效。

---

## 目录结构

```
weekly-report-lark/
├── SKILL.md                       # Claude Code 使用的 Skill 定义（面向 AI）
├── README.md                      # 人类可读的项目说明（你正在阅读）
├── config.example.json            # 配置模板（复制到 ~/.weekly-report-config/）
├── project-mapping.example.json   # 项目映射模板（复制到 ~/.weekly-report-config/）
├── LICENSE                        # 开源协议
└── examples/
    ├── weekly-report-example.md   # 脱敏后的周报输出示例
    └── config-example.json        # 完整配置示例
```

运行时，AI 会按以下优先级查找配置文件：

1. `~/.weekly-report-config/config.json`
2. `~/.weekly-report-config/project-mapping.json`
3. 回退到 skill 目录下的同名文件

---

## License

MIT © [maxliux5](https://github.com/maxliux5)
