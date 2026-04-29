# 🚀 Facebook 跨境广告与舆情智能协同 Agent (AI Workspace)

![Python](https://img.shields.io/badge/Python-3.9%2B-blue)
![Discord.py](https://img.shields.io/badge/Discord.py-2.0%2B-purple)
![Facebook Graph API](https://img.shields.io/badge/Facebook_Graph_API-v19.0-blue)
![LLM Driven](https://img.shields.io/badge/AI_Agent-GPT--4o-green)

> **⚠️ 声明 (Disclaimer)**：本项目涉及公司核心投放风控策略与 API 密钥，当前仓库仅作为 **架构展示 (Showcase)** 与 **技术文档**，移除了核心业务逻辑与敏感配置。如需观看完整实机运行效果，请参阅演示视频或联系作者。

## 📖 项目简介 (Introduction)

本项目是一套部署于海外 Ubuntu 环境的 **基于大模型驱动的 Facebook 广告自动化盯盘与舆情风控智能体 (Agent)**。
系统通过无缝对接 Facebook Graph API 与 Discord，构建了完整的 **“数据感知 -> AI 推理 -> Human-in-the-loop (人机协同) 决策 -> 闭环执行”** 链路。彻底解决了跨境出海团队中“盯盘疲劳导致无效空耗”与“多语种恶评拉低转化率”的两大核心痛点。

---

## 🛠️ 核心双 Agent 架构 (Core Agents)

### 1. 💰 财务风控 Agent (Financial Risk Control)
传统定时脚本往往会漏算“已关停”广告的消耗，导致财务对账偏差。本 Agent 实现了绝对精准的跨层级账单合并与动态红绿线预警：

* **全局双维对账引擎**：绕过官方分页限制，独立拉取 `Account Level` 总账与 `Ad Level` 活跃账单，确保大盘“今日总消耗/总成效”分毫不差。
* **极值阻击 (斩仓与扩量)**：动态计算 CPA/ROAS。触碰“高耗无单”红线即在 Discord 推送 🛑关停 警报；捕捉到“低价多单”信号即推送 🚀扩量 提示。
* **Discord 实时动态大屏**：基于 Discord Embeds 构建了“异常报警板”与“全局财务概览板”，全天候实时动态刷新。

> **📸 系统实机截图演示：**
> *(  ： `![大盘看板](https://github.com/eileenmittal982-maker/FB-Ads-AI-Agent-Showcase/blob/main/a2a82fa99cf6bc1f3bce54174d630068.png)` )*
> `[(https://github.com/eileenmittal982-maker/FB-Ads-AI-Agent-Showcase/blob/main/086a3e13877897977c8c357614cc4086.png)]`
> `[(https://github.com/eileenmittal982-maker/FB-Ads-AI-Agent-Showcase/blob/main/8132184bd195efb444032ac62b8af4a9.png)]`

### 2. 🛡️ 舆情安保 Agent (Public Relations Security)
高并发下，爆款贴文常混入极难识别的南亚/中东俚语脏话。本 Agent 充当了 24 小时无休的超级客服：

* **多语种意图推理**：利用 LLM 强大的语境理解能力，精准捕捉拼音缩写与多语种变体辱骂。
* **Agent 智能切片并发 (Smart Chunking)**：面对瞬间涌入的海量评论，采用动态切片（每批 30 条）策略喂入大模型，杜绝 Token 溢出与 JSON 断裂。
* **超时自动托管 (Auto-Fallback)**：发现恶评后，系统会在 Discord 渲染带有 `[确认删除]`, `[确认回复]`, `[忽略]` 的交互视图。若人工 5 分钟内未干预，Agent 将接管最高权限，闭环调用 FB API 自动清理。

> **📸 系统实机截图演示：**
> `(https://github.com/eileenmittal982-maker/FB-Ads-AI-Agent-Showcase/blob/main/1b4b894772ef8411f0329d7543cb2a77.png)]`

---

## ⚙️ 技术亮点 (Tech Highlights)

为应对海外复杂的网络波动，系统在底层做了极强的鲁棒性建设：

1. **多重网络防御装甲**：针对 FB API 常见的 `ConnectionResetError` 与 `Rate Limit`，内置带退避算法的异步重试机制（Async Retry），保障长期后台运行不死机。
2. **状态机与黑屋隔离**：基于本地 JSON 缓存实现了死号隔离机制。对于因为支付失败/封禁的账户，系统会提取具体死因置顶报警，并提供 Discord 一键 `[✅ 已处理]` 按钮将其移入黑名单，释放系统算力。
3. **异步并发处理**：基于 `asyncio` 和 `discord.ext.tasks`，让 API 抓取、AI 判定与 UI 渲染在不同的事件循环中并行跑通。

### 💻 核心逻辑展示 (Code Snippet)
*为展示架构能力，此处摘录 Discord 交互视图 (Human-in-the-loop) 的底层实现逻辑：*

```python
# Discord UI 交互与 Agent 超时接管逻辑示例 (已脱敏)
class ActionView(discord.ui.View):
    def __init__(self, action_data, real_token):
        super().__init__(timeout=300) # 5分钟人工干预窗口
        self.action_data = action_data
        self.real_token = real_token
        self.is_handled = False

    async def on_timeout(self):
        # 核心：人工未操作时，Agent 超时自动接管
        if self.is_handled: return
        if self.action_data['action'] == 'delete':
            await execute_delete_comment(self.action_data['comment_id'], self.real_token)
            # 更新面板状态...

    @discord.ui.button(label="🔴 确认删除", style=discord.ButtonStyle.danger)
    async def confirm_delete(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.defer()
        # 人工介入，调用 FB API 执行删除动作
        success = await execute_delete_comment(self.action_data['comment_id'], self.real_token)
        await self.disable_all_buttons(interaction)
        await interaction.followup.send("✅ 👤 已删除！" if success else "❌ 删除失败。")
