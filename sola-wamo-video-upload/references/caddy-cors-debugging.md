# Caddy CORS 白屏问题排查

## 症状
网站 HTTP 200 正常，终端 curl 正常，但浏览器全白屏（多人多浏览器均如此）。

## 根因
`index.html` 中的 `<script type="module" crossorigin src="/assets/main-XXXXXX.js">` 包含 `crossorigin` 属性。
带这个属性的模块请求，浏览器要求响应携带 `Access-Control-Allow-Origin` CORS 头。
Caddy 未配置此头 → 浏览器拦截加载 → React 无法初始化 → 白屏。

## 错误修复方案（不要用）

```bash
# ❌ 错误：新建 @assets matcher + handle 块
@assets {
    file
    path /assets/*
}
handle @assets {
    cors *
}
```

原因：`handle @assets` 块会优先匹配 `/assets/*` 路由，但文件在 `srv` 根之外，
Caddy 在该块内找不到文件，返回 404 而不是交给后面的 `file_server` 处理。

实测：加了此配置后，所有 `/assets/*` 返回 404（即使文件存在）。

## 正确修复方案

在**原有的** `handle` 块内加 header 指令，不新建块：

```
handle /assets/* {
    header Access-Control-Allow-Origin *
    file_server
}
```

注意：是修改现有的 `handle` 块，不是新建。

## 回滚步骤

若已推送错误的 Caddyfile：

```bash
# 1. 用上次备份的旧版还原
scp -i ~/.ssh/windows_backup/id_rsa /tmp/Caddyfile.old ubuntu@wamo.city:/home/ubuntu/apps/Caddyfile

# 2. 重载 Caddy
ssh -i ~/.ssh/windows_backup/id_rsa ubuntu@wamo.city "cd /home/ubuntu/apps && caddy reload"
```

## 验证方法

修复后，在浏览器 DevTools → Network 中：
1. 找到 `main-XXXXXX.js` 的响应头
2. 确认有 `Access-Control-Allow-Origin: *`
3. 确认 JS 文件 HTTP 200（非 404）

或用 curl 本地模拟：
```bash
curl -sI "https://sola.wamo.city/assets/main-Bu2-hSW4.js" | grep -i "access-control"
```