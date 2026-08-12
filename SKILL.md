---
name: multi-ai-crosscheck
description: 通过浏览器 MCP 用原生登录态访问对话 AI，就同一问题分别提问，抓取各自答案后做交叉比对，输出带置信度的最终结论。三源固定为豆包、DeepSeek、千问。当用户说「交叉验证」「多个AI对比」「问问豆包和DeepSeek」「多家AI比对」「验证一下这个说法」或需要跨信源核实事实性结论时使用。严禁直连任何 AI 厂商 API。
agent_created: true
---

# 多源 AI 交叉验证

用浏览器 MCP 驱动用户已登录的 Chrome，向多个对话 AI 提同一个问题，抓回答案做交叉比对，给出带置信度的结论。

## 红线原则（不可协商）

1. **只走浏览器 UI，绝不直连接口。** 不得调用任何厂商的 HTTP API、不得复用 cookie/token 发请求、不得用 fetch/XHR 模拟对话接口。所有交互必须是「在页面上填输入框 → 点发送按钮 → 读页面 DOM」。违反会导致账号封禁。
2. **只读取页面已渲染的内容。** 允许用 `chrome_javascript` 读 DOM、做轮询等待，禁止用它发起任何网络请求。
3. **节制并发。** 三站串行发送，站间间隔 ≥1 秒。不做批量轰炸、不做自动化循环刷题。
4. **不碰用户已有会话。** 每次都开新对话，不在用户的历史会话里追加消息。

## 前置检查（必做，否则会静默失败）

**Chrome 窗口必须真实可见，不能处于最小化状态。**

这是实测踩到的最大的坑：Chrome 被最小化时 `document.visibilityState === 'hidden'`，此时
- **DeepSeek 会静默丢消息** —— 会话被创建、标题也生成了，但消息列表完全不渲染，`.ds-markdown` 数量为 0，用户消息也看不见，等于白发一次。
- 豆包、千问受影响较小，但同样不可靠。

而且这是 Chrome 内核对最小化窗口冻结 rAF 导致的，**JS 层面覆写 `document.hidden` / `visibilityState` 无效**（试过，vis 能改成 visible，但消息依然不渲染）。PowerShell 还原窗口的方式会被安全策略拦截。

所以流程开始前必须：

```js
// chrome_javascript 在任一目标 tab 执行
return JSON.stringify({vis: document.visibilityState, focus: document.hasFocus()});
```

若 `vis !== 'visible'`，**停下来直接告诉用户**：「请把 Chrome 窗口点出来（不要最小化），保持在前台，我再继续。」不要硬跑。

## 工程约束

- `chrome_javascript` 单次执行 **默认 16 秒超时**，MCP 层还有约 **30 秒硬超时**。轮询等待必须分段：单次调用内 `setTimeout` 累计不超过 **12 秒**，靠多次调用接力，不要写长 while 循环。
- `chrome_read_page` 返回的 `ref_x` 会过期，**不要跨调用复用**。定位优先用 CSS 选择器。
- 各站类名大量是构建哈希（如 DeepSeek 的 `_27c9245`），随版本变。**优先用语义化选择器**（id、aria-label、稳定语义类名），哈希类名只做兜底。
- 用 `chrome_navigate({newWindow: true})` 新建窗口会抢焦点，**导致其他窗口的 tab 变 hidden**。所以三站要放在**同一个窗口**里，靠 `chrome_switch_tab` 切换，不要给每站开独立窗口。

## 站点适配表（豆包/DeepSeek/千问 均已实测）

详见 `references/site-profiles.md`。核心速查：

| 站点 | 新对话 URL | 输入框 | 发送按钮 | 答案容器 |
|---|---|---|---|---|
| 豆包 | `https://www.doubao.com/chat/` | `textarea.semi-input-textarea` | `button#flow-end-msg-send` | `[class*="message-list"]`（需裁剪） |
| DeepSeek | `https://chat.deepseek.com/` | `textarea[placeholder*="发送消息"]` | `div[role="button"].ds-button--primary` | `.ds-markdown`（取最后一个） |
| 千问 | `https://chat.qwen.ai/` | `textarea.message-input-textarea` | `button.send-button` | `.response-message-content.phase-answer`（取最后一个） |

`chrome_fill_or_select` 能正确触发 React 的 onChange（三站实测发送按钮都会由灰变亮），不需要手动派发 input 事件。

## 源集合（固定三源）

流程固定使用 **豆包、DeepSeek、千问** 三个源，不写死也不动态增减。交叉比对里的共识/采纳计数用 **3** 做分母（如 `[3/3]`、`[2/3]`），不要用其他数字。

## 高风险话题通用约束

医疗/用药/诊断/疫苗、金融/税务/投资等**高风险或强时效性**话题：
- 结论置信度**不得高于「中」**。
- 最终结论必须加一句「以上仅为 AI 参考，请以执业医师 / 官方指南 / 权威信源为准」。
- 涉及具体金额、剂量、政策的，额外建议用户去官网或专业渠道再核一次。

> 一句话：三源交叉验证提升的是「覆盖面与共识强度」，对高风险话题它只能降低不确定性，不能替代专业判断。

## 执行流程

### 1. 准备
- `get_windows_and_tabs` 看现有 tab。三站若已开着就复用，缺哪个补哪个，**都放同一个窗口**。
- 做前置可见性检查。
- 确认用户三站都是登录态（页面上能看到用户名/会话列表即可，不要去读 cookie）。

### 2. 逐站发送（串行）
对每个站点：
1. `chrome_switch_tab` 切到该 tab
2. 导航到新对话 URL（用 `chrome_javascript` 执行 `location.href = '...'` 比 navigate 更不容易抢焦点）
3. 等页面就绪：轮询输入框是否存在
4. `chrome_fill_or_select` 填入问题原文（**三站问题必须完全一致**，不要各自改写，否则比对失去意义）
5. `chrome_click_element` 点发送
6. 间隔 ≥1 秒再处理下一站

### 3. 等待与抓取
**完成判定用「文本稳定法」**，跨站通用、抗改版：连续 3 次采样（间隔 1.2 秒）答案文本长度和内容都不变 → 视为生成完毕。

分段轮询模板（单次 ≤12 秒）：

```js
const SEL = '.ds-markdown';                       // 换成对应站点的答案容器
const pick = () => { const m = [...document.querySelectorAll(SEL)]; return m.length ? m[m.length-1].innerText : ''; };
let prev = window.__xc_prev || '', stable = window.__xc_stable || 0;
for (let i = 0; i < 8; i++) {                      // 8 × 1.2s ≈ 9.6s，安全
  await new Promise(r => setTimeout(r, 1200));
  const cur = pick();
  if (cur && cur === prev) { stable++; if (stable >= 3) break; } else { stable = 0; }
  prev = cur;
}
window.__xc_prev = prev; window.__xc_stable = stable;
return JSON.stringify({ done: stable >= 3, len: prev.length, text: prev.slice(0, 4000) });
```

`done` 为 false 就再调一次，接力直到 true。总等待超过 3 分钟仍未稳定 → 记为「该源超时」，不阻塞其他源。

辅助信号（加速判定，非必需）：
- 千问：生成中发送按钮是 `button.send-button.disabled`
- DeepSeek：深度思考模式下第一个 `.ds-markdown` 是思考过程，**答案取最后一个**
- 豆包：`message-list` 文本里会混入用户问题（开头）和推荐追问（结尾），需裁剪

### 4. 文本清洗
- 豆包：去掉开头的用户问题行，去掉结尾以问号结尾的推荐追问行
- DeepSeek：若开启深度思考，剔除思考块，只留最终答案
- 千问：`.phase-answer` 已经是纯答案，去掉「已经完成思考」前缀即可
- 三站统一：去掉「内容由 AI 生成，请仔细甄别」之类的免责声明

### 5. 交叉比对与输出

默认采用**分歧标注模式**（用户偏好，不要擅自改成多数表决静默出单一答案）：

先把各源答案拆成可比对的**事实断言**，再逐条归类（n = 3）：

- **共识点**：≥2 家表述一致的断言。全部命中标注 `[3/3]`，部分命中标 `[k/3]`（如 `[2/3]`）
- **分歧点**：各家说法冲突的，逐条列出「豆包说 X / DeepSeek 说 Y / 千问说 Z」，**不要藏起来**
- **独有信息**：只有一家提到的关键内容，标注来源

置信度评级：

| 等级 | 判据 |
|---|---|
| 高 | 多数源（≥2/3）在核心结论上一致，无实质冲突 |
| 中 | 接近对半分，或多数一致但都缺乏具体依据；高风险/强时效话题的上限 |
| 低 | 结论互斥，或涉及时效性强、易过期、个体差异大的信息 |

**最终必须明确写出推荐答案和理由**，不能只罗列分歧让用户自己猜。若判断不了，明说「此题需要权威信源核实，各源均不可靠」并给出建议的核实途径。高风险话题（医疗/用药/金融等）必须在结论里加一句「以上仅为 AI 参考，请以执业医师 / 官方指南 / 权威信源为准」。

输出模板：

```
## 结论（置信度：高/中/低）
<一段话给出推荐答案 + 为什么这么判断>

## 共识点
- [3/3] xxx
- [k/3] xxx（豆包、千问一致，DeepSeek 未提及）

## 分歧点
1. 关于 xxx
   - 豆包：...
   - DeepSeek：...
   - 千问：...
   - 倾向：<哪个更可信，理由>

## 各家独有
- DeepSeek 额外提到：...

## 原始答案
<折叠或按需展示各源原文>
```

## 降级与异常

| 情况 | 处理 |
|---|---|
| 某站未登录（跳登录页） | 跳过该源，告知用户「X 站需要重新登录」，**绝不代填账号密码** |
| 某站超时/报错 | 记为超时，用剩余源继续，结论里注明「仅 k 源」并降一档置信度 |
| 只剩 1 源可用 | 明确告知「无法交叉验证，以下是单源答案」，置信度直接判低 |
| 触发验证码/风控 | **立即停止全部自动化**，告知用户手动处理，不要重试 |
| 各源答案严重互斥 | 置信度判低，建议用户做一轮反诘验证（把冲突点回抛给各源追问依据） |

## 配套工作台

`workbench/ai-crosscheck-workbench.html` 是配套的记录台（随仓库一同发布），用于沉淀问题队列、各源答案对照和结论归档，支持豆包/DeepSeek/千问三源。跑完一轮后可提示用户把结果录入归档。
