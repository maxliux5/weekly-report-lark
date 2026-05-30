---
name: weekly-report-lark
description: Use when the user asks to generate or update a weekly report, weekly summary, standup recap, or work log from Lark/Feishu tasks, calendar events, meeting notes, docs, or chat messages.
---

# Weekly Report From Lark Evidence

Use this skill to collect weekly work evidence through `lark-cli`, classify that evidence by project and ownership strength, then write a reusable weekly report.

## Core Principles

Weekly reports are not a writing problem first. They are a **context collection** problem first.

Do not start by drafting prose.
Start by collecting evidence from work traces:

- tasks
- calendar events
- meeting notes and meeting todos
- chat messages
- recently edited docs

Then classify. Then summarize.

The most valuable weekly report often comes from **invisible work**: decisions made in chat, small technical clarifications, unblockings, handoffs, incident follow-ups, and alignment work that never became a formal task. Find those traces, but keep the final report grounded and professional.

Never include raw private messages, secrets, access tokens, IP allowlists, internal incident details, or unnecessary personal names in the final report. Paraphrase work evidence into safe project-level bullets.

## Prerequisites Check

Before using this skill, verify that `lark-cli` is installed and available:

```bash
which lark-cli
lark-cli --version
```

If `lark-cli` is not found, stop and ask the user to install it first. Do not proceed without a working CLI.

Then verify authentication status:

```bash
lark-cli auth status
```

If not authenticated or scopes are insufficient, prompt the user to run:

```bash
lark-cli auth login --domain calendar,task,vc,drive,docs
lark-cli auth login --scope "search:message search:docs:read"
```

## Configuration (Optional)

All configuration is optional. The only hard requirement is a working `lark-cli`.

If configuration files exist, read them in this priority:
1. `~/.weekly-report-config/config.json` — user-specific config (preferred)
2. `./config.json` — skill directory config (fallback)
3. `~/.weekly-report-config/project-mapping.json` — user-specific mappings (preferred)
4. `./project-mapping.json` — skill directory mappings (fallback)

The config file may contain:

- `weekly_report_doc_name` — target document name
- `weekly_report_doc_url` — target Feishu document URL
- `output_format` — `feishu` | `markdown` | `plain`
- `auto_write_back` — whether to append to the doc automatically
- `ownership_labels` — customizable labels for the three ownership buckets

If no config exists, use these defaults:
- `weekly_report_doc_name` → `"周报汇总"`
- `output_format` → `"feishu"`
- `auto_write_back` → `true`
- `ownership_labels` → Chinese defaults (我主导/负责的, 我参与评审/同步的, 后续需要我跟进的)

Read `project-mapping.json` if it exists. Use the keyword-to-project mappings to guide project grouping when evidence titles or content contain those keywords.

## Before You Start

Read these first:

- `../lark-shared/SKILL.md`
- `../lark-task/SKILL.md`
- `../lark-calendar/SKILL.md`
- `../lark-vc/SKILL.md`
- `../lark-im/SKILL.md`
- `../lark-doc/SKILL.md`
- `../lark-workflow-standup-report/SKILL.md`
- `../lark-workflow-meeting-summary/SKILL.md`

This skill only supports **user identity**.

Do not use bot identity for weekly reports. Bot identity usually cannot see the user's personal tasks, calendar, private docs, or chat history correctly.

Minimum likely auth:

```bash
lark-cli auth login --domain calendar,task,vc,drive,docs
lark-cli auth login --scope "search:message search:docs:read"
```

## When To Use

Typical triggers:

- “帮我写这周周报”
- “整理一下我这周做了什么”
- “从飞书里拉素材生成周报”
- “更新周报文档”
- “把这周内容追加到周报总文档”

Do not use this skill when:

- the user only wants a polished rewrite of an already-written weekly report
- the user only wants one single source summarized and does not need a weekly report

## Time Range

Default time range:

- **current week, Monday 00:00 to now**

If the user clearly says “上周”, “过去 7 天”, or gives explicit dates, use that instead.

Always compute dates explicitly before running commands. Do not rely on vague natural-language date strings in CLI flags.

## Default Output Target

This skill is optimized for a **long-lived personal weekly report document** in Feishu.

Default behavior:

1. collect evidence
2. generate the weekly section
3. ensure a target document exists (create one if needed)
4. append or insert the new section into the weekly report document

The target document is determined as follows:

- If `config.json` exists and specifies `weekly_report_doc_url`, use that document.
- If no config exists or no `weekly_report_doc_url` is set, **automatically create a new Feishu document** using `lark-cli docs +create` (or equivalent command), name it `"周报汇总"` (or the configured `weekly_report_doc_name`), and use it as the target.
- After creating the document, save its URL back to `~/.weekly-report-config/config.json` so future runs reuse it.

If the target document is unavailable and creation fails, or if `auto_write_back` is `false`, fall back to returning structured Markdown in chat first.

## Data Sources

Use sources in this order. Do a lightweight pass across the main sources, including chat, then stop deeper collection when evidence is sufficient. Do not collect noise just because a source exists.

### 1. Tasks

Tasks are the cleanest structured signal for:

- completed work
- in-progress work
- due items

Preferred command:

```bash
lark-cli task +get-my-tasks --page-all --format json
```

Task lists can include old backlog items. After fetching, filter locally:

- keep tasks created, updated, completed, or due inside the requested range
- keep stale tasks only when another source this week references the same topic
- ignore personal, vague, or old backlog items that have no current-week signal
- do not treat an old incomplete task as work completed this week

### 2. Calendar / Standup Signals

Use this to infer:

- what coordination work happened
- what time was spent in reviews / syncs / planning
- what was scheduled but possibly not completed

Preferred command:

```bash
lark-cli calendar +agenda --start "<ISO8601>" --end "<ISO8601>"
```

### 3. Meeting Notes And Meeting Todos

Use meeting search + notes when meetings are likely a major part of the week:

```bash
lark-cli vc +search --start "<YYYY-MM-DD>" --end "<YYYY-MM-DD>" --format json --page-size 30
lark-cli vc +notes --meeting-ids "id1,id2,..." --format json
```

Use meeting notes to extract:

- decisions made
- follow-up actions
- blockers surfaced
- meeting todos that point to the user or to the user's owned projects

When `vc +notes` returns `note_doc_token`, fetch the note body before summarizing:

```bash
lark-cli docs +fetch --api-version v2 --doc "<note_doc_token>" --doc-format markdown --format json
```

If `--doc-format text` fails for meeting notes, retry with `markdown`. If note fetching still fails, fall back to the meeting title, shared doc titles, todos, and related chat messages.

Do not turn every meeting into a weekly-report bullet. Only keep meetings that changed work.

### 4. Chat Messages

Use message search because important progress often lives only in chat.

Preferred approach:

1. find relevant chat(s)
2. search messages from the current user in the requested range
3. paginate exhaustively

Example:

```bash
lark-cli im +chat-search --query "<chat keyword>" --format json
lark-cli im +messages-search --query "" --chat-id <chat_id> --sender <current_user_open_id> --start "<ISO8601>" --end "<ISO8601>" --page-size 50 --page-all --format json
```

Rules:

- Always prefer `--format json`
- Always prefer `--page-all` for report tasks
- Narrow with `--chat-id`, `--sender`, `--start`, `--end` before paginating
- Group by topic, not by raw chronology
- Use meeting titles, project names, doc titles, and `project-mapping.json` keywords to find relevant chats
- Run at least one sender-limited message search for weekly reports; do not skip chat solely because tasks or meetings exist
- Treat private chats as supporting evidence only when clearly work-related
- Do not quote raw chat unless the user explicitly asks; paraphrase into work-safe bullets

High-signal chat evidence includes:

- the user made or clarified a technical decision
- the user defined an interface, API contract, schema, event, state machine, or handoff rule
- the user coordinated owners, next steps, risk handling, or release timing
- the user debugged, unblocked, or followed up on an incident
- the user pointed to a concrete doc, PR, dashboard, test result, or runbook

### 5. Recently Edited Docs

Use doc search to find documents edited in the requested period.

Preferred command:

```bash
lark-cli docs +search --as user --query "" --filter '{"sort_type":"EDIT_TIME"}' --page-size 20 --format json
```

Rules:

- get the current user's `userOpenId` from `lark-cli auth status`
- filter by `edit_user_id == userOpenId`
- prefer docs whose `update_time` is inside the requested period
- use recent docs as evidence of direct ownership, not just awareness
- if a doc was only opened or mentioned but not edited by the user, do not count it as direct output
- if the user owns a doc but another person edited it this week, count it as project context unless other evidence shows direct user contribution

When a relevant doc needs to be cited in the final report, use its real Feishu URL so the final document renders as clickable linked text.

## Collection Workflow

Follow this order:

1. determine date range
2. collect tasks
3. collect calendar context
4. collect meetings and meeting todos
5. fetch meeting note bodies when note tokens exist
6. collect sender-limited chat messages from relevant chats
7. collect recently edited docs
8. merge evidence by project
9. classify each item by ownership strength
10. draft the weekly report
11. ensure target document exists (read from config; if missing, create via `lark-cli docs +create` and persist URL to config)
12. write back to the weekly report doc unless the user asked for chat-only output

## Ownership Classification

Never jump straight from raw logs to polished prose.

First classify each project item into one of these buckets:

- **我主导/负责的**
- **我参与评审/同步的**
- **后续需要我跟进的**

### Classification Rules

Put an item into **我主导/负责的** only when there is evidence such as:

- the user directly edited the related doc
- the user owns the follow-up action
- the user drove alignment, planning, testing, rollout, or integration
- the user produced a concrete artifact, schedule, report, or technical clarification
- the user's chat messages define a technical contract, decision, integration path, incident diagnosis, or handoff rule that others rely on

Put an item into **我参与评审/同步的** when:

- the user attended the meeting
- the user gave input or joined review
- the user learned progress for context
- there is no evidence that the user owned the resulting output

Put an item into **后续需要我跟进的** when:

- the meeting todo points to the user
- the user is clearly responsible for the next step
- the work is not complete yet but should remain visible in the weekly report

Do **not** write a meeting-attendance item as direct output.

If the user only attended a content-creation or strategy meeting and did not own resulting work, write it as participation or omit it if it is low signal.

When evidence conflicts, choose the weaker bucket and mention uncertainty briefly in the draft.

## Project Grouping Rules

The weekly report should be organized by **project**, not by source and not by chronological event order.

Examples of project grouping:

- 前端架构 / 组件库建设
- 后端服务 / API 开发
- AI 功能接入 / 模型对接
- 技术文档 / 知识沉淀
- 稳定性 / 监控告警

Rules:

- merge tasks, meetings, docs, and messages into the same project section
- do not repeat the same work item under multiple projects
- if a meeting spans multiple projects, only keep the project-relevant part
- if a project has only passive context and no actionable consequence, keep it brief or omit it
- consult `project-mapping.json` for keyword → project mappings when evidence titles or content contain those keywords

## Writing Rules

The final weekly report should feel like a human wrote it, but it must stay grounded in evidence.

Required rules:

- do not invent work that has no supporting signal
- do not over-claim based on one vague chat message
- do not write “推进了” for something that was only a meeting the user attended
- separate direct ownership from participation
- mention blockers and follow-ups clearly
- compress repetitive low-signal tasks
- turn chat evidence into outcome-oriented bullets: "clarified X", "defined Y", "coordinated Z", "unblocked W"
- remove sensitive implementation details unless they are necessary and safe for the target weekly report audience
- keep the tone plain and professional

## Default Output Structure

Use this structure unless the user requests another format.

The section titles for ownership buckets must be read from `config.json` → `ownership_labels`:
- `lead` → e.g. "我主导/负责的"
- `participate` → e.g. "我参与评审/同步的"
- `follow_up` → e.g. "后续需要我跟进的"

If `config.json` is missing, use the Chinese defaults above.

```md
# <date-range>

## 项目一：<project name>
### <ownership_labels.lead>
- ...
- 相关文档：[文档 A](https://...)

### <ownership_labels.participate>
- ...

### <ownership_labels.follow_up>
- ...

## 项目二：<project name>
...
```

## Feishu Write-Back Rules

When writing back to the weekly report doc:

- keep the document in reverse chronological order
- add the newest weekly section near the top
- keep historical sections intact
- use Feishu links for related docs instead of plain text titles
- avoid dumping raw URLs in standalone lines when linked text is possible

If the user asks for document update:

- prefer `docs +update --api-version v2 --as user`
- preserve the existing weekly-report structure
- do not overwrite unrelated historical content

## Compression Rules

When there are too many raw items:

- merge same-topic updates into one bullet
- collapse operational chores into one grouped bullet
- keep only meetings with decisions or action items
- ignore routine status chatter
- prefer one strong project bullet over three weak source bullets

## Troubleshooting

| Symptom | Resolution |
|---------|------------|
| `lark-cli: command not found` | Stop. Ask the user to install lark-cli and ensure it is on PATH. Do not fake data. |
| `auth required` / 401 errors | Prompt the user to re-authenticate with the Minimum likely auth commands above. |
| No target doc configured | Automatically create a new Feishu doc via `lark-cli docs +create`, name it "周报汇总", and persist the URL to config for future reuse. |
| Doc write fails (403/404) | Fall back to chat-only output. Preserve the structured Markdown so the user can copy-paste manually. |
| Doc search returns nothing | If `search:docs:read` scope is missing, continue with tasks + calendar + meetings + messages. |
| Meeting notes are empty | Some meetings lack auto-transcription. Use calendar events and related chat messages as fallback evidence. |
| Meeting note fetch fails | Retry `docs +fetch --api-version v2` with `--doc-format markdown`; if it still fails, use note metadata and shared docs only. |
| Task list is too noisy | Filter tasks locally by requested date range and corroborate stale tasks with meetings, docs, or messages. |
| Chat search is too noisy | Search relevant chats first, then restrict by `--sender`, `--chat-id`, `--start`, and `--end`; summarize by topic instead of dumping messages. |
| Ownership classification is ambiguous | Default to the weaker bucket. Prefer "participate" over "lead" when evidence is inconclusive. |
| Project grouping is inaccurate | Refer to `project-mapping.json` mappings. If no keyword matches, use AI inference but err on the side of broader project names. |

## Fallback Behavior

If one source fails:

- continue with the remaining evidence sources
- state clearly what evidence was missing
- avoid over-claiming certainty

If doc search is unavailable because `search:docs:read` is missing:

- continue with tasks + calendar + meetings + messages
- do not infer direct authorship from meeting mentions alone

If the default weekly report doc is unavailable:

- return a structured weekly draft in chat
- keep the same project + ownership structure
