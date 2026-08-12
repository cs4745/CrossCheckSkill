# 站点适配详表

实测环境：Chrome + streamable-mcp-server（Chrome 扩展形态），2026-08-07 验证通过。
类名带哈希的（如 `_27c9245`、`content-KTJ1Rj`）随发版会变，**发现失效时优先重新探测，不要硬改哈希**。

探测脚本（通用，粘到 `chrome_javascript` 里跑）：

```js
const out = { url: location.href, vis: document.visibilityState };
out.inputs = [...document.querySelectorAll('textarea,[contenteditable="true"]')]
  .map(e => ({ tag: e.tagName, id: e.id, cls: String(e.className||'').slice(0,60),
               ph: e.placeholder || e.getAttribute('data-placeholder') || '' }));
out.buttons = [...document.querySelectorAll('button,div[role="button"]')]
  .filter(b => { const r = b.getBoundingClientRect(); return r.width > 0 && r.height > 0; })
  .map(b => ({ tag: b.tagName, id: b.id, cls: String(b.className||'').slice(0,45),
               aria: b.getAttribute('aria-label')||'', dis: b.disabled === true,
               txt: (b.innerText||'').trim().slice(0,12) }));
return JSON.stringify(out);
```

先填入文本再跑一次，对比哪个按钮从 disabled 变 enabled —— 那个就是发送按钮。

---

## 豆包 doubao.com

| 项 | 值 |
|---|---|
| 新对话 URL | `https://www.doubao.com/chat/` |
| 输入框 | `textarea.semi-input-textarea`（placeholder：`发消息或按住空格说话...`） |
| 发送按钮 | `button#flow-end-msg-send` ← **稳定 ID，三站里最可靠的** |
| 答案容器 | `[class*="message-list"]` |
| 单条内容块 | `[class*="content-"]`（哈希，如 `content-KTJ1Rj`） |

页面用 Semi Design（字节自研）+ Tailwind + radix。按钮 id 形如 `radix-:rr:` 的是弹层触发器，别误点。

**答案裁剪**：`message-list` 的 innerText 结构是

```
用一句话说明什么是 CAP 定理          ← 用户问题
CAP 定理：分布式系统中，...           ← 真正的答案
CAP定理的具体应用场景有哪些？          ← 推荐追问（要去掉）
如何在分布式系统中实现一致性？
一致性、可用性、分区容错性三者之间的关系是什么？
```

按行切分，去掉首行（等于用户问题）和结尾连续的问句行。

**实测样本**：
> CAP 定理：分布式系统中，一致性 (C)、可用性 (A)、分区容错性 (P) 三者无法同时完全满足，必须舍弃 C 或 A 其一。

注意：豆包发完消息后 URL 会变成 `https://www.doubao.com/chat/<会话ID>`。

---

## DeepSeek chat.deepseek.com

| 项 | 值 |
|---|---|
| 新对话 URL | `https://chat.deepseek.com/` |
| 输入框 | `textarea[placeholder*="发送消息"]`（类名 `_27c9245` 是哈希，别用） |
| 发送按钮 | `div[role="button"].ds-button--primary`（注意是 div 不是 button） |
| 答案容器 | `.ds-markdown`，取 **最后一个** |
| 深度思考开关 | 输入框下方「深度思考」，开启后第一个 `.ds-markdown` 是思考链 |
| 模式标签 | 页面顶部显示「快速模式」/「深度思考」 |

**最大的坑：窗口最小化时静默丢消息。**

实测现象：Chrome 最小化（`visibilityState === 'hidden'`）时点发送
- 会话会被创建，URL 跳到 `/a/chat/s/<uuid>`
- 侧边栏会出现新会话，标题也正确生成了（说明内容确实到了服务端）
- **但消息列表完全空白**，`document.querySelectorAll('.ds-markdown').length === 0`，连自己发的那条用户消息都不渲染
- `document.body.innerText` 只有侧边栏内容（约 1600 字符）
- 刷新、重新点会话、新开窗口加载该会话，**都救不回来**，这个会话就是空的

已排除的解法：
- ❌ 覆写 `document.hidden` / `visibilityState` + 派发 `visibilitychange`：属性能改成 visible，消息照样不渲染（Chrome 内核冻结 rAF，JS 改不了）
- ❌ `chrome_navigate({background:false})`：返回 "Activated existing tab" 但窗口没真正前台化
- ❌ `chrome_switch_tab`：只切 tab，不能把最小化的窗口还原
- ❌ PowerShell `Add-Type` + `ShowWindow` / `WScript.Shell.AppActivate`：被安全策略拦截

**唯一可靠解法：让用户手动把 Chrome 窗口点出来。** 跑之前先检查 `visibilityState`，不是 visible 就停下来提示。

对照验证：同一个 tab 加载**历史会话**时，`.ds-markdown` 能正常渲染出 10 个块 —— 证明选择器没问题，纯粹是发送时机的可见性问题。

---

## 千问 chat.qwen.ai

| 项 | 值 |
|---|---|
| 新对话 URL | `https://chat.qwen.ai/` |
| 输入框 | `textarea.message-input-textarea`（placeholder：`有什么我能帮您的吗？`） |
| 发送按钮 | `button.send-button`（`aria-label="发送"`） |
| 生成中状态 | 发送按钮变 `button.send-button.disabled` |
| 答案容器 | `.response-message-content.phase-answer`，取最后一个 ← **最干净** |
| 外层容器 | `.chat-response-message`（会多带「已经完成思考」前缀） |
| markdown 内层 | `.custom-qwen-markdown` / `.qwen-markdown` |
| 新建对话 | `[aria-label="新建对话"]` |
| 模型选择器 | `[aria-label="Select Model"]`（实测显示 Qwen3.8-Max） |

千问类名是语义化的，三站里最稳定，基本不用担心改版。

**实测样本**：
> CAP定理指出，在一个分布式系统中，一致性（Consistency）、可用性（Availability）和分区容错性（Partition tolerance）这三个基本特性最多只能同时满足其中的两个，永远无法三者兼顾。

发完消息 URL 变成 `https://chat.qwen.ai/c/<uuid>`。

---

## 三源答案差异观察（CAP 定理样例）

| 源 | 表述 | 差异点 |
|---|---|---|
| 豆包 | 「三者无法同时完全满足，必须舍弃 C 或 A 其一」 | 更准确 —— 点明了 P 在分布式系统中不可放弃，实际只能在 C/A 间取舍 |
| 千问 | 「最多只能同时满足其中的两个，永远无法三者兼顾」 | 教科书式表述，但没说明 P 通常是必选项 |
| DeepSeek | （本轮未取得） | — |

这个样例正好说明交叉验证的价值：两家都"对"，但豆包的表述更贴近工程实践。**比对时要看的不只是结论对错，还有完备度和精确度。**
