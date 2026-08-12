# CrossCheckSkill · 面向 Agent 的自动安装指引

> 本文件是「写给 Agent 看的安装说明书」。把本文件（或其中内容）直接发送给任意支持 skills 的 Agent（如 WorkBuddy），它会自主完成检测、下载、部署与校验，无需你逐步指挥。

---

你是被指派安装 **CrossCheckSkill** 的 Agent。请严格按下方步骤自主执行，**仅在「必须人工介入」的环节停下询问用户**，其余一律自动完成并在最后汇总报告。

## 0. 适用范围
- 适用框架：WorkBuddy 等以目录形式管理 skills 的 Agent（技能目录约定为 `<skills_root>/<skill-name>/SKILL.md`）。
- 不适用：无 skills 机制、或无法访问本地文件系统的纯对话 Agent（请改用仓库 README 的手动安装说明）。

## 1. 前置核验（不满足立即提示，不要硬装）
1. **Chrome MCP 必须已配置。** 本技能完全依赖 Chrome MCP 驱动的浏览器自动化，没有它技能无法运行。检测方法：列出当前可用工具，若存在 `chrome_*` / `mcp__*chrome*` 系列工具即视为已就绪；否则**停止并提示用户**按 https://github.com/hangwin/mcp-chrome/blob/master/README_zh.md 完成安装（扩展 + 桥接进程），装好后再继续。
2. 确认你拥有本地文件系统读写能力（能创建目录、写入文件）。

## 2. 定位技能目录（自适应）
按以下优先级确定安装根目录，命中即停：
1. 环境变量 `WORKBUDDY_SKILLS`（若存在）。
2. 用户级：`~/.workbuddy/skills/`。
   - Windows 实际路径：`$env:USERPROFILE\.workbuddy\skills\`（即 `C:/Users/<user>/.workbuddy/skills/`）
   - macOS / Linux：`~/.workbuddy/skills/`
3. 项目级（若用户指定了工作区）：`<workspace>/.workbuddy/skills/`
4. 以上都不存在则创建用户级目录。
目标技能目录固定为 `<skills_root>/multi-ai-crosscheck/`。

## 3. 获取源码（自适应取数）
优先用 git：
- `git clone --depth 1 https://github.com/cs4745/CrossCheckSkill.git <临时目录>`
若 git 不可用，降级用 WebFetch / 直接下载抓取以下原始文件到本地：
- SKILL.md：`https://raw.githubusercontent.com/cs4745/CrossCheckSkill/main/SKILL.md`
- references/site-profiles.md：`https://raw.githubusercontent.com/cs4745/CrossCheckSkill/main/references/site-profiles.md`
- workbench/ai-crosscheck-workbench.html：`https://raw.githubusercontent.com/cs4745/CrossCheckSkill/main/workbench/ai-crosscheck-workbench.html`

## 4. 部署
将下列内容写入 `<skills_root>/multi-ai-crosscheck/`：
- `SKILL.md`（核心，必须）
- `references/site-profiles.md`（必须）
- `workbench/ai-crosscheck-workbench.html`（配套记录台，可选但建议）
若已存在同名文件则覆盖（确保拿到最新版）。

## 5. 校验
1. 确认 `<skills_root>/multi-ai-crosscheck/SKILL.md` 存在，且首行 / frontmatter 含有效 skill 标识。
2. 确认 `references/site-profiles.md` 存在。
3. 扫一遍 `SKILL.md` 与 `references/site-profiles.md`，确认**不含「阿福」**（历史四源版本残留）——本技能当前为三源（豆包 / DeepSeek / 千问）。
4. 若用 git，记录 `git -C <临时目录> rev-parse HEAD` 作为版本号。

## 6. 完成报告（给用户）
简要输出：
- 安装路径：`<skills_root>/multi-ai-crosscheck/`
- 版本：`HEAD <hash>` 或「文件抓取于 <时间>」
- 是否已配置 Chrome MCP：是 / 否（否则附配置链接）
- 使用提示：触发语「交叉验证一下」「多个 AI 对比」等，使用前先手动打开豆包 / DeepSeek / 千问登录页。

## 7. 边界与兜底
- 不要修改 skill 内部逻辑去「适配」你的框架；它已经是跨平台目录式结构，只需放到正确目录即可被加载。
- 若你所在 Agent 的 skills 机制特殊（如需要额外 `manifest.json` / `metadata`），先按原生机制放置 `SKILL.md`，再按需补一个最小 manifest，并告诉用户你补了什么。
- 任何需要浏览器扩展授权、重启 Agent 的步骤，明确交给用户，不要假装已完成。
