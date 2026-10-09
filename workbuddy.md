# workbuddy.md —— WorkBuddy 开发过程记录

> 本文件为麦当劳程序员节创意开发大赛联动活动（WorkBuddy 专项奖励）提交材料：记录本项目使用 WorkBuddy 开发时的对话上下文与工作流程，用于核验是否符合联动活动奖励条件。本文件为该会话的结构化忠实记录。

---

## 1. 项目信息

| 项目 | 内容 |
|---|---|
| 项目名称 | McdBirthdayButler（麦麦生日会管家） |
| 项目类型 | WorkBuddy Skill（基于麦当劳 MCP mcd-mcp） |
| 开发工具 | WorkBuddy（AI 智能体客户端，GLM 模型驱动） |
| 开发日期 | 2026 年 10 月 9 日 |
| 开发平台 | Windows（win32） |

## 2. 开发起因：用户原始需求（对话上下文摘录）

用户在 WorkBuddy 对话中提出（2026-10-09 14:51）：

> 你是一位精通麦当劳 MCP 的全栈开发助手，请帮我开发一个名为 "McdBirthdayButler（麦麦生日会管家）" 的 Skill，帮助家长规划孩子在麦当劳举办的生日派对。
>
> 【核心用户与场景】家长想给 3–10 岁的孩子办麦当劳生日会，但不知道哪些门店能办、有什么场次、怎么收费。用户只需说"孩子 X 岁、Y 个小朋友、预算 Z 元、想要 A 月 B 日、在 C 城区附近"，Skill 就能完成全流程规划。
>
> 【必须使用的麦当劳 MCP 工具与调用链】
> 1. mcp__mcd-mcp__query-party-city：确认目标城市是否支持派对业务；
> 2. mcp__mcd-mcp__query-party-store：查询该城市可办生日会的门店列表；
> 3. mcp__mcd-mcp__query-party-store-date：按门店查询指定月份的可预约日期；
> 4. mcp__mcd-mcp__query-party-store-session：查询指定日期的具体场次与剩余容量；
> 5. mcp__mcd-mcp__party-order-create：在用户明确确认后创建派对订单；
> 6. 辅助：mcp__mcd-mcp__now-time-info（判断日期合法性）、mcp__mcd-mcp__calculate-price（核验价格）。
>
> 【交互要求】信息不足时一次性追问缺项；推荐 2–3 个候选方案用表格对比；每个方案给出"管家点评"；创建订单前必须展示完整订单摘要并获得用户明确确认；全程语气亲切但高效。
>
> 【交付物】Skill 定义文件（含名称、描述、触发词）、系统提示词、README.md（含安装方法与使用示例）、MCP_INTEGRATION.md，并按照大赛仓库要求组织文件夹、附 .gitignore。

## 3. WorkBuddy 开发工作流程（对话过程结构化记录）

### 阶段一：规范与资料调研（14:51–14:53）

WorkBuddy 在同一轮并行完成了两件事：

1. **加载 `skill-creator` 技能**：获取 Skill 创建官方规范——SKILL.md 必须含 YAML frontmatter（name/description，`agent_created: true`）、渐进式披露设计（SKILL.md 主流程 + references/ 详细手册）、使用祈使句书写等要求。
2. **抓取大赛仓库** `https://github.com/M-China/mcd-developer-innovation-challenge`：确认参赛项目的强制文件清单——README.md、CONTEST_DECLARATION.md（文件名及内容均不可改动）、MCP_INTEGRATION.md、源代码（形式不限），以及 workbuddy.md（参加 WorkBuddy 专项奖励时必须提交）。

### 阶段二：官方文件与 MCP 工具链核实（14:53–14:55）

1. **逐字抓取官方参赛声明原文**（`CONTEST_DECLARATION.md` raw 文件），确保本地文件与官方版本一字不差，满足"内容不可改动"要求。
2. **抓取麦当劳 MCP Server 官方文档**（`https://github.com/M-China/mcd-mcp-server`）：确认接入地址 `https://mcp.mcd.cn`、Streamable HTTP 协议、Bearer Token 鉴权、600 次/分钟限流、401/429 错误码，以及 v1.0.7（2026-07-29）新增派对主题活动工具族的事实。
3. **加载本机已连接 mcd-mcp 的真实工具 schema**（通过 WorkBuddy 工具检索，7 个工具）：`query-party-city`、`query-party-store`、`query-party-store-date`、`query-party-store-session`、`party-order-create`、`now-time-info`、`calculate-price`。

**关键发现（本项目的核心技术决策）**：官方文档未明示、但真实 schema 中明确的两条前置约束——
- 所有派对查询工具都要求先经 `mall-points-products` / `mall-product-detail` 获取派对主题商品的 `spuId`（下单还需 `skuId`，且仅 `shopId=5` 的商品支持下单）；
- `party-order-create` 的 `partyType` 区分包场(1)/拼团(2)，商品 `partyType=-1` 时必须询问用户后确认。

该发现被固化为 SKILL.md 工作流的 Step 0（"锁定派对商品"）与 MCP_INTEGRATION.md 的调用链图，避免实际使用中出现无效下单。

### 阶段三：Skill 全量编写（14:55–14:57）

WorkBuddy 按大赛文件清单逐文件落盘（Agent 模式直接写入工作区）：

| 文件 | 内容要点 |
|---|---|
| `SKILL.md` | frontmatter（name/description/触发词）+ 8 步工作流系统提示词（锁定派对商品 → 一次性追问 → 日期校验 → 城市 → 门店 → 日期/场次 → 方案表格+管家点评 → 订单确认单红线 → 下单后交付）+ 交互规则（数据忠实、限流意识、失败兜底）+ 边界合规 |
| `references/playbook.md` | 渐进式披露的详细手册：开场承接分档话术、预算不足/日期全满/城市不支持/年龄超范围/人数不符等异议处理、管家点评语料库、下单复查清单（禁止事项） |
| `CONTEST_DECLARATION.md` | 官方原文照抄，零改动 |
| `README.md` | 项目介绍、目标用户表、MCP Token 申请与 WorkBuddy/其他 Client 安装方法、4 个使用示例（一句话启动/方案对比/一次性追问/订单确认单） |
| `MCP_INTEGRATION.md` | Server 概况、9 个 Tools 一览、ASCII 调用链流程图、参数速查表（每个参数标注来源工具）、业务价值矩阵、6 条安全与工程约束 |
| `.gitignore` | 密钥凭证排除（.env/token/mcp.json.local 等，对应参赛声明"不含真实 Token"要求）+ 常规忽略项 |

### 阶段四：校验与打包（14:57–14:59）

1. 运行 skill-creator 官方脚本 `package_skill.py` 自动校验，一次通过：**`✅ Skill is valid!`**（frontmatter 格式、描述质量、目录结构均符合规范）。
2. 首次打包因输出目录放在 Skill 内部导致 zip 自包含，发现后改用外部目录 `mbb_dist/` 重新打包，并用 Python `zipfile` 逐项核对包内 6 个文件清单无误。

### 阶段五：补充提交材料（14:59，本阶段）

用户要求补交 `workbuddy.md`，WorkBuddy 基于当次开发会话生成本结构化记录并置于项目根目录，随后重新打包使发行 zip 同步包含。

## 4. 产出物清单

```
mcd-birthday-butler/
├── SKILL.md                  # Skill 定义 + 系统提示词（核心交付物）
├── references/playbook.md    # 话术手册（渐进加载）
├── README.md                 # 项目介绍 / 安装 / 使用示例 / 目标用户（大赛必交）
├── MCP_INTEGRATION.md        # MCP 调用流程与业务价值（大赛必交）
├── CONTEST_DECLARATION.md    # 参赛声明，官方原文（大赛必交，内容不可改动）
├── workbuddy.md              # 本文件：WorkBuddy 开发过程记录（WorkBuddy 专项奖励必交）
├── .gitignore
└── （发行包：mcd-birthday-butler.zip，含以上全部文件）
```

## 5. 使用 WorkBuddy 的价值说明

1. **规范自动对齐**：WorkBuddy 主动加载 skill-creator 技能，使 SKILL.md 一次通过官方校验脚本，无需手工排查格式问题。
2. **事实核验闭环**：同一会话内完成大赛规则、参赛声明原文、MCP 官方文档、真实工具 schema 四层资料核实，纠正了"派对查询可直接发起"的常见误解，避免提交后因调用链错误被驳回。
3. **并行效率**：调研、抓取、多文件写入大量并行执行，从用户提出需求到通过校验打包仅约 8 分钟。
4. **可直接联调**：WorkBuddy 会话内已连接 mcd-mcp 连接器，Skill 开发完成后可立即在同一环境中实测"城市→门店→日期→场次→确认下单"全链路。
