---
name: notion-class-reminder
description: 从 Notion 课程表生成上课提醒话术，并写入话术库的流程
version: 1.0.0
author: shiyu
platforms: [linux, macos, windows]
prerequisites:
  env_vars: [NOTION_API_KEY]
triggers:
  - "生成上课提醒"
  - "5/27 上课提醒"
  - "课程表话术"
---

# Notion 上课提醒话术流程

从课程表读取场次信息 → 套用话术模板 → 写入 Notion 话术库

## 流程概览

1. 读课程表（AI Agent 线上共学主页）→ 找到目标场次的日期、时间、主题、讲师
2. 读嘉宾介绍页（课程主页里的嘉宾表格）→ 找对应嘉宾的简介
3. 套用"上课提醒"话术模板（话术库根页 → 上课提醒 toggle）
4. 将结果写入话术库：上课提醒 toggle 下新建日期 toggle，内容放里面

---

## Step 1：读课程表

课程表 Notion 页面，表格结构（5列）：
`日期 | 开始和结束时间 | 课程内容 | 讲师 | 录制回放 & PPT`

课程表根页 ID：`8e73890a64b68204b60281e87c62ed12`

```bash
# 先拿到表格 block ID（表格 has_children=true）
curl -s "https://api.notion.com/v1/blocks/8e73890a64b68204b60281e87c62ed12/children" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03"
# 找 type=table 的 block，记下其 id
```

```bash
# 读表格内容（table block 的 children 即 table rows）
curl -s "https://api.notion.com/v1/blocks/{table_block_id}/children" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03"
```

---

## Step 2：读嘉宾介绍

嘉宾表格在课程主页里，结构：
`姓名 | 方向 | 简介 | 分享日期 | 主题`

课程主页（AI Agent 线上共学 | 22天...）block 结构：
- 嘉宾 table block：`has_children=true`，读其 children

```bash
curl -s "https://api.notion.com/v1/blocks/{guest_table_block_id}/children" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03"
```

---

## Step 3：话术模板

### 格式判断规则

根据课程表"课程内容"列的文本形态判断用哪个模板：

| 特征 | 模板 |
|---|---|
| 课程内容含"第一课/第二课/第三课"等系统课程名，或大纲用✨分隔 | 模板 B 普通上课提醒 |
| 课程内容是主题描述 + 讲师有专属嘉宾介绍 | 模板 A 嘉宾提醒 |
| 讲师为 Shing 或"开放麦" | 模板 B |
| 讲师为外部嘉宾（乱码/jiang/孙圆圆等） | 模板 A |

---

### 模板 A - 嘉宾提醒
```
今晚八点，关于"主题"线上分享！AI Agent 共学教室不见不散~
嘉宾：
姓名1，简介1
姓名2，简介2
时间：月/日 HH:MM
会议链接：https://meeting.tencent.com/dm/xxx
会议号：XXX-XXX-XXX
```

**多嘉宾格式规范：**
- "嘉宾："单独一行，下面每人一行
- 人名和简介之间用**逗号**，不用冒号
- 复制发送时每行无多余空格

### 模板 B - 普通上课提醒
```
今晚八点，学习"主题"相关知识！AI Agent 共学教室不见不散~
✨ 要点1
✨ 要点2
...
时间：月/日 HH:MM
会议链接：...
腾讯会议号：XXX-XXX-XXX
```

---

## Step 4：写入话术库

话术库结构：
- 根页：`话术`（ID: `36c3890a64b680afb7e1e62c7437aead`）
  - 话术集（toggle）
  - 报名后（toggle）
  - **上课提醒**（toggle）→ 目标父级
    - **5/27**（toggle）→ 新建这里
      - 内容段落（paragraph）

```bash
# 1. 在上课提醒下创建日期 toggle
curl -s -X PATCH "https://api.notion.com/v1/blocks/{上课提醒block_id}/children" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{
  "children": [
    {
      "object": "block",
      "type": "toggle",
      "toggle": {
        "rich_text": [{"type": "text", "text": {"content": "5/27", "link": null}}],
        "color": "default"
      }
    }
  ]
}'
# 返回的 results[0].id 即新建 toggle 的 block_id

# 2. 在日期 toggle 下写入内容段落
curl -s -X PATCH "https://api.notion.com/v1/blocks/{新toggle_block_id}/children" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{
  "children": [
    {"object": "block", "type": "paragraph", "paragraph": {"rich_text": [...]}},
    ...
  ]
}'
```

---

## 相关 Notion 页面 ID

| 页面 | ID |
|---|---|
| 话术（话术库根） | `36c3890a64b680afb7e1e62c7437aead` |
| 上课提醒（toggle） | `36c3890a-64b6-80e5-8987-fe4d5d802fef` |
| 课程表 | `8e73890a64b68204b60281e87c62ed12` |
| AI Agent 线上共学主页 | `36d3890a64b680668c39d881ee7211f2` |

---

## 注意事项

- 嘉宾介绍若在课程表表格中找不到，需额外读取嘉宾介绍页
- Ren 等嘉宾若不在嘉宾表格中，需用户直接提供简介
- 多嘉宾格式：人名和介绍之间用逗号，嘉宾二字单独一行
- 写入前确认目标 toggle 路径：话术 → 上课提醒 → 日期