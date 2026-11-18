# AI-Gateway-updata

AI Gateway（Windows / WebView2 版）的**免服务器更新源**。

客户端不连任何后端，只读本仓库根目录的 `update.json`（raw CDN）。
整个更新链路：客户端拉清单 → 比对版本 → 命中则弹强制更新窗 → 点按钮跳夸克网盘下载。

## update.json 字段

| 字段 | 必填 | 说明 |
|---|---|---|
| `version` | ✅ | 最新版本号。客户端 `0.9.5` 看到 `0.9.6` 即判定有更新 |
| `minVersion` | — | 低于此版本的客户端**强制**更新（不弹「稍后」）。留空或等于 `version` 均可 |
| `force` | — | `true` 时所有旧版本都强制。与 `minVersion` 任一命中即强制 |
| `downloadUrl` | ✅ | 下载页地址。当前固定为夸克网盘分享链接 |
| `downloadName` | — | 网盘里的文件名，弹窗里会显示，方便用户对照 |
| `publishedAt` | — | 发布日期，纯展示 |
| `notes` | — | 更新说明，原样显示在弹窗正文 |

## 发新版流程

1. 把新版打进 `AI-Gateway-Windows-<版本>.zip` 上传夸克网盘（链接可长期复用）
2. 改 `update.json` 的 `version` / `downloadName` / `notes`
3. commit + push

客户端最长 6 小时一次后台检查；用户在概览页点「检查更新」会绕过缓存立刻拉取。
`raw.githubusercontent.com` 有几分钟 CDN 同步延迟，等了没生效就用「检查更新」硬刷。

## 版本比较规则

按 `.` 分段逐段比数值（`0.9.10` > `0.9.9`），段数不等时缺的补 0；
`-` / `+` 之后的预发布与构建元数据不参与排序。

## 拉取失败的处理

没网 / GitHub 不可用 / JSON 写坏了，客户端一律**放行**（不弹窗、不拦截），
只把 `downloadUrl` 作为普通提示展示。更新检查不能成为把用户挡在门外的理由。

## 强制更新到底强制什么（发版前必读）

`force: true` 或版本低于 `minVersion` 时，客户端不是"多弹一个窗口"，而是：

| 层 | 行为 |
|---|---|
| 网关端口 | **完全不监听**。客户端（Cursor / Claude Code…）直连 `127.0.0.1:8790/v1` 得到 `Connection refused` |
| 管理面 | `POST /ui/gateway/start` 返回 `ok:false`「必须更新才能启动网关」 |
| 界面 | 整页被遮罩 + `inert` 锁死；侧栏、按钮、表格点不到，Tab 键盘也进不去 |
| 出口 | 只有「退出应用」和「立即更新」两个按钮 |

闸门一旦落下，**在同一进程内不会解除**；用户必须下载新版、覆盖安装、重启客户端才能恢复。
所以 `force` 是一个重开关：

- 只想提示、允许用户稍后再说 → `force: false`，只靠 `minVersion` 卡"太老必须升"的下限
- 想彻底停掉旧版 → `force: true`

实测证据（预置强制缓存后启动的临时实例）：

```
日志  [aigw-server] 强制更新闸门已落，网关不启动：强制更新 已安装 0.9.7 → 最新 0.9.8
state running=false  gateClosed=true
netstat  无网关端口
curl   Connection refused
POST /ui/gateway/start → {"ok":false,"message":"必须更新才能启动网关（当前 0.9.7，0.9.8 已发布）"}
```

## 本地发版清单

```bash
# 1. 打包（在 aigw-windows 目录）
powershell -File build-web.ps1

# 2. 传夸克网盘（链接可长期复用）

# 3. 改 APP_VERSION（不改的话新装版本会一直提示更新）
#    server/src/main/kotlin/dev/aigw/server/EngineHost.kt

# 4. 改本仓库 update.json 的 version / downloadName / notes，commit + push
```
