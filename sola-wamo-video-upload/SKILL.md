---
name: sola-wamo-video-upload
description: 将本地视频和逐字稿上传到 wamo.city 共学网站服务器，按课程表驱动，只上传课程表中有对应的条目。
---

# 视频+逐字稿上传服务器 Workflow

## 触发条件
用户要求将本地视频和逐字稿上传到共学网站服务器。

## ⚠️ 最高前提：不破坏原网页内容

> **入口规则**：操作 `shing19/openclaw-cn-tutorial` 任何路径前，必须先读仓库根目录 AGENTS.md，再按其指引去读对应子目录的 AGENTS.md。

**操作规则源**：
- 仓库根 AGENTS.md：https://github.com/shing19/openclaw-cn-tutorial/blob/main/AGENTS.md
- 站点 AGENTS.md：`courses/social-layer/openclaw-colearning-site/AGENTS.md`

每次任务开始前，重新读取这两个文件（不要凭上次会话的旧上下文）。

## 文件发现

本地源目录：`C:\Users\lenovo\Downloads\`

命名规则：
- 视频：`录制_MMDD.mp4`（例：`录制_0526.mp4`）
- 逐字稿：`录制_MMDD.txt`（例：`录制_0527.txt`）
- `MMDD` = 两位月 + 两位日，不足前面补 0
- 两个文件同名不同后缀，成对出现

从文件名提取 `MMDD` → 去课程表匹配对应日期的活动。

## 前端静态发布流程（AGENTS.md 第 94-110 行）

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
SITE_DIR="$REPO_ROOT/courses/social-layer/openclaw-colearning-site"
cd "$SITE_DIR"
npm run build
rsync -az --exclude 'openclaw-graduation-site/' dist/ ubuntu@wamo.city:/home/ubuntu/apps/sola-wamo-city/
```

**只改静态资源时，不需要重启 API 或 Caddy。**

## ⚠️ 源码修改铁律

> "改课程表里的课件、回放、资料链接时，除了改 src/App.jsx，还要检查相关 manifest 和真实文件，避免按钮有了但页面打不开。"（AGENTS.md 第 182-184 行）

**严禁**绕过源码直接修改服务器 bundle。正确流程：
1. 修改本地源码文件
2. `npm run build`
3. 部署前自检
4. rsync 部署

## 域名与缓存警告

- `sola.wamo.city` → 直接回源，服务最新 JS ✅
- `wamo.city` → 经过 Cloudflare CDN，可能缓存旧版 JS ❌

**每次验证必须用 `https://sola.wamo.city`**，不是 `wamo.city`。

## 回放链接规则（AGENTS.md 第 186-208 行）

课程表回放按钮指向 `/replay/<slug>`，需要三处同时修改：

| 位置 | 对象 | 缺少时的后果 |
| --- | --- | --- |
| `src/replayManifest.js` 的 `REPLAYS` | slug 配置 | "找不到这个回放" |
| `src/App.jsx` 的 `li["s2-{key}"]` | 回放元数据 | 课程表按钮不出现 |
| `src/App.jsx` 的 `di["s2-{key}"]` | 回放页面常量表 | 白屏：`Cannot read properties of undefined` |
| `src/App.jsx` 的 `Gi` 课程表资源列 | 按钮 | 无回放按钮 |

**四者缺一不可。**

---

## Step 1：读 AGENTS.md 确认规则

```bash
# 仓库根 AGENTS.md
curl -fsSL https://raw.githubusercontent.com/shing19/openclaw-cn-tutorial/main/AGENTS.md

# 站点 AGENTS.md（本地）
REPO_ROOT=$(git rev-parse --show-toplevel)
SITE_DIR="$REPO_ROOT/courses/social-layer/openclaw-colearning-site"
cat "$SITE_DIR/AGENTS.md"
```

每次任务开始前重新读，不要凭上次会话的旧上下文。

---

## Step 2：确认本地文件

```bash
# 列出所有录制文件
ls -lh "/mnt/c/Users/lenovo/Downloads/录制_"*.mp4 2>/dev/null
ls -lh "/mnt/c/Users/lenovo/Downloads/录制_"*.txt 2>/dev/null
```

从文件名提取 `MMDD`（如 `0526`）→ 确认对应日期的课程表条目。

---

## Step 3：上传到服务器

> ⚠️ 速度警告：WSL → wamo.city 上传 MP4 约 20-50 KB/s（100MB 需 30-60 分钟）。建议后台 rsync + `notify_on_complete`。

### txt 逐字稿（小文件）—— 直接 rsync

```bash
rsync -av -e "ssh -i ~/.ssh/windows_backup/id_rsa" \
  "/mnt/c/Users/lenovo/Downloads/录制_MMDD.txt" \
  ubuntu@wamo.city:/home/ubuntu/apps/sola-wamo-city/replays/
```

### MP4 视频（大文件）—— 后台 rsync 传

```bash
terminal(background=True) {
  "rsync -av --progress -e 'ssh -i ~/.ssh/windows_backup/id_rsa' \
    \"/mnt/c/Users/lenovo/Downloads/录制_MMDD.mp4\" \
    ubuntu@wamo.city:/home/ubuntu/apps/sola-wamo-city/replays/"
}
```
传完后 Hermes 会通知你。

---

## Step 4：在服务器重命名

命名规则：
- 视频：`s2-{主题前缀}-{MD5前8位}.mp4`
- 逐字稿：`s2-{主题前缀}.txt`

```bash
ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city \
  "ls -lh /home/ubuntu/apps/sola-wamo-city/replays/ | grep -E 's2-|\.mp4|\.txt' | tail -20"
```

找到刚上传的文件，用 `mv` 重命名。MP4 文件需先计算 MD5 前 8 位：

```bash
# 本地计算 MD5 前 8 位
md5sum "/mnt/c/Users/lenovo/Downloads/录制_MMDD.mp4" | cut -c1-8
# 例：abc12345
```

重命名：
```bash
ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city \
  "cd /home/ubuntu/apps/sola-wamo-city/replays && \
   mv '录制_MMDD.mp4' 's2-{主题前缀}-abc12345.mp4' && \
   mv '录制_MMDD.txt' 's2-{主题前缀}.txt'"
```

---

## Step 5：确认 slug 并分析课程表

从 replayManifest.js 和 App.jsx 分析缺少回放的条目：

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
SITE_DIR="$REPO_ROOT/courses/social-layer/openclaw-colearning-site"
cd "$SITE_DIR"

# 已有 replay slug
grep -o '"s2-[^"]*":' src/replayManifest.js | tr -d '":' | sort

# 服务器上已有的视频文件
ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city \
  "ls /home/ubuntu/apps/sola-wamo-city/replays/*.mp4" | xargs -n1 basename
```

对比课程表（Gi 数组）+ replayManifest.js REPLAYS + 服务器文件，找出缺少回放的日期。

---

## Step 6：修改源码

> ⚠️ **只改本地源码，不改服务器 bundle**。

### 6a. 确认仓库干净，origin/main 最新

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
SITE_DIR="$REPO_ROOT/courses/social-layer/openclaw-colearning-site"
cd "$SITE_DIR"

git fetch origin
git status --short
git log HEAD..origin/main --oneline
```

必须无未 commit、未 add，origin/main 最新。

### 6b. 修改 src/replayManifest.js（添加 REPLAYS 条目）

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
SITE_DIR="$REPO_ROOT/courses/social-layer/openclaw-colearning-site"

# 查看当前 REPLAYS 结构
grep -n 's2-' "$SITE_DIR/src/replayManifest.js"
```

在 `REPLAYS` 对象中添加新条目：

```javascript
// src/replayManifest.js 的 REPLAYS 对象格式
"REPLAYS": {
  "s2-opening": { title: "...", videoUrl: "/replays/s2-opening-xxxxxxxx.mp4", ... },
  // 新增：
  "s2-{slug}": {
    title: "标题",
    subtitle: "副标题",
    speaker: "主讲人",
    date: "M.DD（周） HH:MM-HH:MM",
    videoUrl: "/replays/s2-{slug}-{hash}.mp4",
    transcriptUrl: "/replays/s2-{slug}.txt",
  },
}
```

### 6c. 修改 src/App.jsx（三处修改：li 对象、di 常量表、Gi 课程表）

> ⚠️ **必须同时修改三处，缺一不可**。

#### 6c-1. `li["s2-{slug}"]` — 回放元数据

在 `li` 对象中添加新条目（注意逗号分隔）：

```javascript
// 例：在 "s2-guest-share" 条目后添加
,"s2-{slug}":{title:`标题`,subtitle:`副标题`,speaker:`主讲人`,date:`M.DD（周） HH:MM-HH:MM`,videoUrl:`/replays/s2-{slug}-{hash}.mp4`,transcriptUrl:`/replays/s2-{slug}.txt`}
```

#### 6c-2. `di["s2-{slug}"]` — 回放页面常量表（防白屏关键）

在 `di` 对象中添加：

```javascript
"s2-{slug}":{title:`标题`,subtitle:`副标题`,speaker:`主讲人`,date:`M.DD（周） HH:MM-HH:MM`,videoUrl:`/replays/s2-{slug}-{hash}.mp4`,transcriptUrl:`/replays/s2-{slug}.txt`}
```

#### 6c-3. `Gi` 课程表资源列 — 加回放按钮

**保护现有课件按钮**的格式（双层数组）：
```javascript
// 有课件 + 加回放：[[slot,slot],[{label:`打开课件`,href:Fi},{label:`打开回放`,href:ui(`s2-{slug}`)}]]
// 无课件 + 加回放：[,slot,slot,slot,slot,slot,{label:`打开回放`,href:ui(`s2-{slug}`)}]
```

用 grep 找到目标行：
```bash
grep -n "5.26\|Agent 深度" "$SITE_DIR/src/App.jsx" | head -5
```

---

## Step 7：构建 + 验证源码修改

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
SITE_DIR="$REPO_ROOT/courses/social-layer/openclaw-colearning-site"
cd "$SITE_DIR"

npm run build

# 验证三处/四处修改都在 bundle 里
SLUG="s2-{slug}"
echo "=== $SLUG 出现次数（应为 4+） ==="
grep -o "\"$SLUG\"" dist/assets/main-*.js | wc -l

# 对比：其他已有 slug 也还在
for slug in s2-opening s2-inspiration-party s2-qa-session s2-ai-agent; do
  echo -n "$slug: "
  grep -o "\"$slug\"" dist/assets/main-*.js | wc -l
done
```

**判断规则**：
- 新 slug 出现 ≥4 次 ✅
- 已有 slug 没有丢（次数与修改前相当）✅
- 任意已有 slug 次数变 0 → 构建有问题，停止检查

---

## Step 8：部署前自检（AGENTS.md 第 113-141 行）

> ⚠️ **必须逐条做完 4 步，全部通过才能部署**。

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
SITE_DIR="$REPO_ROOT/courses/social-layer/openclaw-colearning-site"
cd "$SITE_DIR"

# 1. 工作树干净，origin/main 最新
git fetch origin
git status --short
git log HEAD..origin/main --oneline
# 必须无未 commit/未 add，origin/main 最新

# 2. wamo 现在跑的 index.html 时间戳
WAMO_MTIME=$(ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city "stat -c %Y /home/ubuntu/apps/sola-wamo-city/index.html")
HEAD_TIME=$(git log -1 --format=%ct origin/main)

# 3. 如果 wamo 比 origin/main 新 → 有人在 commit 之外部署过，ABORT
if [ "$WAMO_MTIME" -gt "$HEAD_TIME" ]; then
  echo "ABORT: wamo index.html ($WAMO_MTIME) newer than origin/main commit ($HEAD_TIME)"
  ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city "cat /home/ubuntu/apps/sola-wamo-city/index.html | grep -oE 'main-[A-Za-z0-9_-]+\.js'"
  exit 1
fi

# 4. dry-run 看会改/删什么
rsync -avz --dry-run --exclude 'openclaw-graduation-site/' \
  dist/ ubuntu@wamo.city:/home/ubuntu/apps/sola-wamo-city/
```

**判断规则**：
- 输出里有 `delete` 字样 → **停止**，确认要删的是什么
- 要删的是 `.mp4` 或 `.txt`（本地 dist/ 里没有）→ 服务器有源码没有的东西 → **先合回源码**
- 只显示 `*.js` / `*.css` 更新 → 安全，继续 Step 9

---

## Step 9：部署

```bash
rsync -az --exclude 'openclaw-graduation-site/' \
  dist/ ubuntu@wamo.city:/home/ubuntu/apps/sola-wamo-city/
```

**不要加 `--delete`**：Vite 产物是 hash 文件名，不删不会被 shadow；加 `--delete` 会删掉服务器上源码没有的视频文件。

---

## Step 10：验证（必须用 `sola.wamo.city`）

```bash
# 生产 bundle 包含所有 slug
JSFILE=$(ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city \
  "grep -o 'main-[^\"]*\.js' /home/ubuntu/apps/sola-wamo-city/index.html | head -1")

for slug in s2-opening s2-inspiration-party s2-qa-session s2-ai-agent s2-{新slug}; do
  echo -n "$slug: "
  ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city \
    "grep -o '\"$slug\"' /home/ubuntu/apps/sola-wamo-city/assets/\$JSFILE" | wc -l
done

# 生产视频文件返回 HTTP 200
for f in s2-{slug}-{hash}.mp4 s2-{slug}.txt; do
  echo -n "$f: "
  curl -sI -o /dev/null -w "%{http_code}" "https://sola.wamo.city/replays/$f"
  echo
done

# API 健康检查
curl -s --max-time 10 "https://sola.wamo.city/api/community/health"
```

**浏览器验证**：
- 清除缓存（`Ctrl+Shift+R`）或用无痕模式
- 访问 `https://sola.wamo.city/#curriculum` → 对应日期行出现"打开回放"按钮
- 访问 `https://sola.wamo.city/replay/s2-{slug}` → 正常播放页，**不是**"找不到这个回放"

---

## Step 11：git commit

用户确认验证通过后，commit 本次相关改动：

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
SITE_DIR="$REPO_ROOT/courses/social-layer/openclaw-colearning-site"
cd "$SITE_DIR"

git add src/replayManifest.js src/App.jsx
git commit -m "feat: add replay for s2-{slug}"
git push origin main
```

只 commit 本次任务相关文件，不要提交无关脏文件。

---

## 常见问题

### Q：课程表里同一个日期有多条活动，回放按钮加在哪一条？
A：同一天多条活动的回放视频和逐字稿是同一个，只加一次回放按钮在该日期的**主要条目**上（如有多个同名日期，用标题关键字区分）。

### Q：上次操作后课件按钮消失了，只剩回放按钮？
A：Gi 课程表资源列格式错误。修复：找到被错误修改的行，恢复为 `[[slot,slot],[{label:`打开课件`,href:Fi},{label:`打开回放`,href:ui(`s2-{slug}`)}]]` 双层数组格式。

### Q：网站没有出现"打开回放"按钮？白屏？
A：
1. 确认访问 `https://sola.wamo.city`（不是 `wamo.city`）
2. `Ctrl+Shift+R` 硬刷新或开无痕模式
3. DevTools → Network 确认加载的 JS 文件名是 index.html 引用的
4. Console 报错 `Cannot read properties of undefined (reading 'category')` → `di` 常量表缺失该 slug
5. 打开回放显示"找不到这个回放" → `replayManifest.js` REPLAYS 缺失该 slug

### Q：rsync --dry-run 显示会删除 .mp4 文件？
A：服务器上有源码没有的视频文件（有人在 commit 之外上传过）。ABORT，先把服务器上的文件对应的 slug 合回源码，再 build + deploy。