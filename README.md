# pi-chrome × CNKI 浏览器控制适配

用 [pi-chrome](https://github.com/tianrendong/pi-chrome) 在 OMP（Oh My Pi / Pi coding agent）里驱动**你自己已登录的 Chrome**，
跑通 CNKI 知网的 **检索 → 摘要 → 下载全文** 闭环。

本仓库是 [`paper-master-4ss`](https://github.com/JingYangYuan/paper-master-4ss) / `paper-lit-4ss` 文献模块 CNKI 阶段后端文档的对外发布版，
由 `scripts/export_pi_chrome_doc.py` 生成，请勿直接编辑。

## 适用场景

- 需要 CNKI 机构登录态、下载权限：pi-chrome 复用你日常 Chrome profile 的 Cookie，不用另开自动化浏览器。
- 需要 Agent 在知网专业检索页填式、提交、筛选、排序、抓摘要、导出 Cookie 并批量下载 PDF。
- 不适用：验证码、passkey/生物识别、原生系统弹窗、跨域 iframe DOM 操作——这些必须人工完成。

## 快速开始

```bash
pi install npm:pi-chrome        # 已装可跳过；会话运行中需 /reload
```

```text
/chrome onboard                 # 显示伴生扩展目录；macOS 会自动打开 chrome://extensions 并复制路径
```

Chrome 侧（一次性，手动）：开启**开发者模式** → **加载已解压的扩展程序** → 选择

```text
~/.omp/plugins/node_modules/pi-chrome/extensions/chrome-profile-bridge/browser-extension
```

```text
/chrome authorize               # 默认 15 分钟；长期用 /chrome authorize indefinite
/chrome doctor                  # 应显示 ✓ Chrome is connected
/chrome revoke                  # 用完撤销
```

## 每次 CNKI 阶段前的四项验收

| 步骤 | 命令 | 通过标准 |
|---|---|---|
| 1 | `chrome_tab action=list` | 返回标签页列表，不报错 |
| 2 | `chrome_tab action=version` | 返回 `extensionVersion` / `bridgeUrl` / `capabilities` |
| 3 | `chrome_navigate`（**不带 target**）到 `about:blank` | 返回 `Navigated to about:blank` |
| 4 | `chrome_evaluate`（**不带 targetId**）读 `location.href` | 返回 `about:blank` |

任一项失败即记录 `浏览器控制不可用` 并停止 CNKI 阶段；先 `/chrome doctor`，再检查伴生扩展是否启用。

## 三条硬约束（2026-09-12 实测，pi-chrome 0.15.51）

1. **不要传 `targetId`**：`chrome_evaluate` / `chrome_snapshot` / `chrome_click` 传 `targetId` 会返回
   `Runtime.evaluate: Detached while handling command`。全流程只走 `chrome_navigate` 建立的单一自动化目标。
2. **页内异步不得 `awaitPromise`**：await 慢 fetch 会中断调用。写成立即返回的 fire-and-forget 挂到
   `window.__x`，再用 `chrome_wait_for { kind: "expression", value: "window.__x !== null" }` 轮询。
3. **点击与读取一律用 `chrome_evaluate`**：kns8s 这类重页面上 `chrome_snapshot` 会
   `inject snapshot script ... timed out after 8000ms`，`chrome_click` 会 `DOM click fallback ... timed out`。
   默认 `hardBackground: true` 下 `chrome_tab activate` 被拒属预期，由用户切到 `Pi Session:` 分组标签完成登录/验证码。

完整失败模式表与恢复步骤见 [pi-chrome-browser.md](pi-chrome-browser.md) §7。

## Cookie：走回环 sink，不进对话

页面不能写文件，而把 `document.cookie` 打印进对话等于把会话凭证写进日志与模型上下文。
OMP 上唯一推荐做法：本机回环 sink 接收一次 POST，0600 落盘，只回报字节数与键数。

```bash
python3 scripts/cnki/cookie_sink.py --out /tmp/cnki_cookie.txt --port 17399 --timeout 240
```

```js
// 页面侧（chrome_evaluate）：立即返回，不 await
fetch('http://127.0.0.1:17399/c', { method: 'POST', body: document.cookie })
  .then(r => r.status)
```

```bash
rm -f /tmp/cnki_cookie.txt     # 下载器用完立即删除
```

## 文件

| 文件 | 内容 |
|---|---|
| [pi-chrome-browser.md](pi-chrome-browser.md) | 完整适配协议：安装授权、目标模型、能力映射、输入层与页内脚本、失败模式表、Cookie 导出、与 ZCode 后端差异、实测记录 |
| [scripts/cnki/cookie_sink.py](scripts/cnki/cookie_sink.py) | Cookie 回环 sink（`--out` / `--port` / `--timeout`；超时 rc=2，非法载荷 rc=3） |

## 归属与许可

- 浏览器控制能力由第三方 MIT 项目 [pi-chrome](https://github.com/tianrendong/pi-chrome) 提供，本仓库只含 CNKI/OMP 侧适配文档与 sink 脚本。
- CNKI 检索式、结果解析、验证码判据、下载与归档属于论文工作流协议，见 [paper-master-4ss](https://github.com/JingYangYuan/paper-master-4ss)。
- 本仓库内容随 `paper-master-4ss` 一同维护，许可 [MIT](LICENSE)。
