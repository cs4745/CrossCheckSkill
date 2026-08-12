# CrossCheckSkill · 多源 AI 交叉验证

通过**浏览器 MCP**驱动你已登录的 Chrome，向多个对话 AI 提同一个问题，抓回答案做交叉比对，输出**带置信度**的最终结论。

> 本技能只走浏览器 UI，绝不直连任何 AI 厂商 API（避免封号）。所有交互都是「在页面上填输入框 → 点发送按钮 → 读页面 DOM」。

## ⚠️ 前置依赖（必看，使用前必须完成）

**本技能依赖 Chrome MCP 驱动已登录的 Chrome，使用前必须提前配置好 Chrome MCP，否则整个交叉验证流程无法运行。**

- Chrome MCP 是一套「浏览器自动化」能力，提供 `navigate / fill_or_select / click_element / javascript / screenshot` 等工具，由 Chrome 扩展 + 本地桥接进程驱动你日常使用的真实 Chrome（保留原生登录态，非无头模式）。
- 技能内部通过 `chrome_*` 系列工具操作页面，没有任何兜底接口：没有 Chrome MCP 就发不出问题、抓不到答案。
- 配置方式请参考官方中文文档（含扩展安装、桥接进程启动、WorkBuddy/MCP 客户端接入）：**https://github.com/hangwin/mcp-chrome/blob/master/README_zh.md**

> 配置完成后建议先手动打开豆包 / DeepSeek / 千问 三个标签页并确保已登录，再触发技能，可避免「首次跳转登录页」打断流程。

## 三源固定集合

| 源 | 地址 | 说明 |
|---|---|---|
| 豆包 | https://www.doubao.com/chat/ | 字节 Semi Design，发送按钮 id 稳定 |
| DeepSeek | https://chat.deepseek.com/ | 窗口最小化会静默丢消息，需前置可见性检查 |
| 千问 | https://chat.qwen.ai/ | 类名语义化最稳定，答案容器最干净 |

交叉比对里的共识/采纳计数固定用 **3** 做分母（如 `[3/3]`、`[2/3]`）。

## 红线原则

1. **只走浏览器 UI，绝不直连接口。** 不调用任何厂商 HTTP API、不复用 cookie/token、不用 fetch/XHR 模拟对话接口。
2. **只读页面已渲染内容。** `chrome_javascript` 仅用于读 DOM 与轮询等待，不发起网络请求。
3. **节制并发。** 三站串行发送，站间间隔 ≥1 秒。
4. **不碰用户已有会话。** 每次都开新对话。

## 关键坑（必看）

**Chrome 窗口必须真实可见（不能最小化）。** 最小化时 `visibilityState === 'hidden'`，DeepSeek 会静默丢消息（会话建了但不渲染）。JS 覆写 `visibilityState` 无效，用脚本还原窗口的方式会被安全策略拦截。**唯一可靠解法：让用户手动把 Chrome 窗口点出来。** 流程前先检查可见性，不是 visible 就停下来提示。

详见 `references/site-profiles.md`（含三站实测选择器、答案容器、裁剪规则、轮询模板）。

## 使用方式

触发语示例：「交叉验证一下」「多个 AI 对比」「问问豆包和 DeepSeek」「验证这个说法对不对」。

技能执行流程：
1. 准备：复用已开 tab，三站放同一窗口；做可见性检查；确认登录态。
2. 逐站串行发送同一问题（问题原文三站必须完全一致）。
3. 文本稳定法轮询抓取答案。
4. 清洗三站答案（去用户问题行、去推荐追问、去思考链、去免责声明）。
5. 分歧标注式交叉比对 + 输出带置信度的结论。

### 置信度评级

| 等级 | 判据 |
|---|---|
| 高 | >=2/3 源在核心结论上一致，无实质冲突 |
| 中 | 接近对半分，或多数一致但缺依据；**高风险/强时效话题上限** |
| 低 | 结论互斥，或涉及时效性强、个体差异大的信息 |

**高风险/强时效话题通用约束**：医疗、用药、金融、税务等，结论置信度不高于「中」，且必须附「以上仅为 AI 参考，请以执业医师 / 官方指南 / 权威信源为准」。

## 配套工作台

`workbench/ai-crosscheck-workbench.html` —— 单文件、零依赖、localStorage 存储的记录台，用于沉淀问题队列、各源答案对照与结论归档。直接浏览器打开即可用，手机电脑自适应。跑完一轮可把结果录入归档。

## 安装

作为 WorkBuddy 技能使用：将本仓库放入技能目录（如 `~/.workbuddy/skills/multi-ai-crosscheck/`），保留 `SKILL.md` 与 `references/` 结构即可。

## 文件结构

    CrossCheckSkill/
    ├─ SKILL.md                      # 技能定义（核心）
    ├─ references/
    │  └─ site-profiles.md          # 三站实测选择器与坑位
    ├─ workbench/
    │  └─ ai-crosscheck-workbench.html  # 配套记录台
    └─ README.md

## 让 Agent 自动安装（零指挥）

本仓库附带一份 **`AGENT_INSTALL.md`**——它是「写给 Agent 看的安装说明书」。你不需要自己一步步操作：

> **直接把 `AGENT_INSTALL.md` 的内容（或本仓库链接）发给任意支持 skills 的 Agent，它会自主完成：**
> 1. 检测当前框架与技能目录（WorkBuddy / 项目级 / 跨平台路径自适应）
> 2. 核验 Chrome MCP 是否已配置（未配置会停下进行提示，附配置链接）
> 3. 用 git 克隆或 WebFetch 降级拉取源码
> 4. 把 `SKILL.md` + `references/` + `workbench/` 部署到正确目录
> 5. 校验文件完整性（含「不含阿福」残留检查）并回报告

即：**发送 → Agent 自适应安装 → 你只需要补「Chrome MCP 扩展授权 / 重启生效」这类必须人工的步骤。** 详见 `AGENT_INSTALL.md`。

## 文件结构

    CrossCheckSkill/
    ├─ SKILL.md                      # 技能定义（核心）
    ├─ references/
    │  └─ site-profiles.md          # 三站实测选择器与坑位
    ├─ workbench/
    │  └─ ai-crosscheck-workbench.html  # 配套记录台
    ├─ AGENT_INSTALL.md             # 面向 Agent 的自动安装指引（发给 Agent 即可装）
    └─ README.md
