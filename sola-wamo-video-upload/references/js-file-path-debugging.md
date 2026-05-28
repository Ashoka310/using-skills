# JS 文件路径排错笔记

## 发现背景（2025-05-27）

用户反馈 5.26 回放上传后，整个网站白屏。Console 报错：

```
Uncaught TypeError: Cannot read properties of undefined (reading 'category')
    at ui (main-Bu2-hSW4.js:55:11729)
    at main-Bu2-hSW4.js:55:25830
```

## 根因：di 回放常量表缺失

### 排查过程

1. 行 55 是 `ui` 函数定义（AI 热点卡片组件，读取 `item.category`）
2. 行 25830 是 `yi` 回放组件里 `o.map(e => (0,M.jsx)(ui, {item:e}, e.id))` — 调用 `ui`
3. 最初怀疑 AI 热点数据异常，实际根因是 **`di` 回放常量表**里缺少 `s2-ai-agent` 条目

### 关键发现

- `grep -c 's2-ai-agent' main-Bu2-hSW4.js` 返回 **1**，但一个完整配置应该有 **≥4 处**：
  - `di` 对象里 1 条（`"s2-ai-agent":{...}`)
  - 课程表按钮里 3 处（5.26 行回放 href、行 25830 的 ui({item:e}) 调用等）
- 原因：之前修复 5.26 课件按钮时，可能覆盖了 `di` 对象的 `s2-ai-agent` 条目

### 验证方法

```bash
# 统计出现次数（应 ≥4）
ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city \
  "grep -o 's2-ai-agent' /home/ubuntu/apps/sola-wamo-city/assets/main-Bu2-hSW4.js | wc -l"

# 确认 di 对象里有该条目（应 =1）
ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city \
  "grep -c '\"s2-ai-agent\":{' /home/ubuntu/apps/sola-wamo-city/assets/main-Bu2-hSW4.js"
```

### 修复

在 `di` 对象中，在 `s2-guest-share` 条目末尾插入新条目（注意加逗号隔开）：

```javascript
"s2-guest-share":{...transcriptUrl:`/replays/s2-guest-share.txt`},"s2-ai-agent":{title:`Agent 深度使用`,subtitle:`如何把 Agent 接进真实项目、复利系统和长期协作`,speaker:`Shing`,date:`5.26（二） 20:00-21:30`,videoUrl:`/replays/s2-ai-agent-3bd1cbf9.mp4`,transcriptUrl:`/replays/s2-ai-agent.txt`}
```

注意：`yi` 组件在 URL `/replay/s2-ai-agent` 时会查找 `di['s2-ai-agent']`，找不到时显示"找不到这个回放"，但白屏的报错是 `ui` 函数的 `category` 报错——这说明是 AI 热点模块的数据异常导致的级联反应。

## 历史记录：双文件不同步（2025-05-27 早前）

服务器 `/home/ubuntu/apps/sola-wamo-city/assets/` 目录存在**两个**内容相近但版本不同的 JS 文件：

| 文件 | 状态 | 说明 |
|------|------|------|
| `main-Bu2-hSW4.js` | ✅ index.html 实际引用 | 当前生效文件 |
| `main-C6o4oT9u.js` | ❌ 旧版（可能已过时）| skill 历史文档一直记录的名字 |

### 验证方法

```bash
# 从 index.html 读取当前实际引用的文件名
ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city \
  "grep -o 'main-[^\"]*\.js' /home/ubuntu/apps/sola-wamo-city/index.html"

# 对比两个文件的 MD5
ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city \
  "md5sum /home/ubuntu/apps/sola-wamo-city/assets/main-*.js" | grep -E 'C6o4oT9u|Bu2-hSW4'
```

## 相关发现

- `wamo.city` 域名有 SSL 握手问题（TLS alert internal error），不能用这个域名验证
- `sola.wamo.city` 直接回源，无 CDN 缓存问题，验证时必须用这个域名
- 视频文件在服务器：`/home/ubuntu/apps/sola-wamo-city/replays/s2-ai-agent-3bd1cbf9.mp4`（248MB）
- 逐字稿文件：`/home/ubuntu/apps/sola-wamo-city/replays/s2-ai-agent.txt`（109KB）