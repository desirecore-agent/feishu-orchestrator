# 飞书编排助手 · 开发与维护指引

> 本文件给**接手维护这个 Agent 的智能体或开发者**看，与 `AGENTS.md` 内容相同（两份必须保持同步）。
> DesireCore 运行时**不读取**本仓库里的这两个文件，所以这里写的是维护指引，不是 Agent 的行为规则——行为规则在 `persona.md` 和 `principles.md`。

---

## 1. 这个仓库是什么

DesireCore 官方市场里「飞书编排助手」Agent 的**内容仓库**。它把自然语言意图翻译成飞书官方 CLI（`lark-cli`）的正确调用，覆盖飞书 23 个业务域 / 512 个 `+shortcut`。

它**不是**飞书命令的说明书——命令目录由飞书官方技能与 `lark-cli schema` 提供，随 CLI 升级更新。这个 Agent 做的是官方技能不覆盖的四件事：

| 增量 | 说明 |
| --- | --- |
| **接入** | 把官方技能接进 DesireCore 的上下文与审批链路，官方分发列表里没有这一环 |
| **纪律** | 官方 CLI 的确认门禁只在退出码里（exit 10），本 Agent 负责把它变成一次真人确认而不是静默重试 |
| **编排** | 跨业务域工作流（站会摘要、会议纪要汇总），官方技能按单产品划分，不跨域 |
| **降级** | 没装 / 没授权 / 没权限 / 租户没开通时如实停下，绝不编造成功 |

## 2. 两个仓库的关系（改动前必须懂）

```
desirecore-agent/feishu-orchestrator   ← 本仓库：Agent 的全部内容
        ↑ 被指向（pin 到具体 commit）
desirecore/market
  └── agents/feishu-orchestrator/
        ├── entry.json                 ← 卡片：写着「内容在本仓库，版本 <SHA>」
        ├── catalog-metadata.v1.json   ← 审核信息（证据六项）
        └── assets/avatar.webp         ← 头像
```

**市场侧只有这三个文件，不能多。** 校验器用 `inline_path.is_file() == pointer_path.is_file()` 判定——`agent.json` 与 `entry.json` **必须恰好存在一个**，都有或都没有都会让条目从市场消失。

**改内容 → 改本仓库；发新版 → 改市场卡片的 pin。** 两者是分开的两步：

| 你要做的 | 动哪里 |
| --- | --- |
| 修文档、调人格、改纪律 | 只改本仓库，推 `main` |
| 让新装的用户拿到新版 | 改市场卡片的 pin（见 §4） |

用户装到的永远是卡片上 pin 的那个 commit，不是本仓库的最新 `main`。这是**有意为之**——上游随便改不会立刻影响已发布的用户，中间有显式审核关口。

## 3. 目录结构与各文件职责

```
agent.json          AgentFS 运行时配置（纯运行时字段，不含市场展示字段）
persona.md          人格（L0/L1/L2 三层）
principles.md       行为规则：L1 Must Do / Must Not / 外部依赖边界 / Priority，L2 详则与升级规则
USAGE.md            使用说明——市场详情页「使用说明」区块渲染的就是这个文件
README.md           仓库说明（给人看）
LICENSE / NOTICE    许可证据，市场 sidecar 的 compliance.licenseEvidencePath 指向 LICENSE
docs/               13 篇业务域文档 + images/ 截图  ⚠️ 见 §5.1，当前没有运行时消费入口
```

**本仓库没有 `skills/` 目录**——这是它与 `dingtalk-workspace` / `invoice-organizer` 最大的结构差异。

### 技能从哪来

`agent.json#required_skills.collections` 声明市场合集 `larksuite-cli` 中的 **23 个**技能，安装 Agent 时由客户端从合集上游取源，落到 Agent 的私有技能目录，带完整 provenance（`providerId` + `sourceRef` + `contentDigest`），可正常更新与卸载。

```json
{"required_skills": {"collections": [{"id": "larksuite-cli", "providerId": "official", "skills": ["lark-shared", "lark-contact", ...]}]}}
```

**必须用 `required_skills`，不能用 `default_enabled.skills`。** 后者只声明「已有技能默认启用」，不负责把技能取过来；两者都写才完整（本仓库两处都有，改一处必须同步改另一处）。`required_skills` 是 DesireCore PR #2587 引入的，`requiredClientVersion` 因此至少是 `10.0.144`——`10.0.143` 的发布 commit 在 #2587 之前合并，不含该能力。

### 合集有 28 个，为什么只声明 23 个

差的 5 个都不是能力缺失：

| 未声明 | 原因 |
| --- | --- |
| `lark-minutes` / `lark-note` / `lark-vc` / `lark-vc-agent` | 官方描述均写明「统一交由 **lark-meeting** 技能处理」，是转发别名壳；`lark-meeting` 已声明 |
| `lark-skill-maker` | 「创建 lark-cli 的自定义 Skill」，开发者元工具，不是终端用户能力 |

**核对合集内容要看远端**：`~/.desirecore/market/official/` 是本机解压副本，可能停在旧 commit（实测见过它只有 27 个技能且不含 `lark-meeting`，据此会误判「声明了不存在的技能」）。以 `desirecore/market` 的 `origin/main` 为准。

## 4. 发新版：pin 要同时改三处

市场卡片 pin 一个 40 位 SHA。**改 ref 必须同时改三处，漏任何一处校验必红**：

| 文件 | 字段 |
| --- | --- |
| `entry.json` | `source.ref` |
| `catalog-metadata.v1.json` | `provenance.content.ref` |
| `catalog-metadata.v1.json` | `governance.compliance.reviewedRef` |

自检别按路径逐个查，**全文扫 40 位 hex** 确认三处一致：

```bash
grep -oE '[0-9a-f]{40}' agents/feishu-orchestrator/*.json | sed 's|.*:||' | sort | uniq -c
# 期望：3 <同一个 SHA>
```

同时更新 `timestamps.reviewedAt.value` 与 `governance.compliance.reviewedAt`（两者必须相等）。

**还有两条容易漏的一致性约束**（都实测撞过 validate 红）：

- `entry.json#license` 是**必填**。写了它就等于声明 `catalog.governance.license.state = known` + `evidencePath`，两边必须对齐
- `entry.json#redistribution` 必须等于 `catalog.governance.redistribution`。迁 pointer 后语义是 `source-pointer-only`（内容不再随 market 分发），catalog 侧旧值若是 `allowed` / `verify-package-terms` 会报 `legacy-consistency`

**顺序**：先推本仓库并用 `git ls-remote` 确认 SHA 可达，**再**改市场卡片。反了会造出指向不存在内容的死卡片。

市场 PR 前跑（`uv` 在 `~/.local/bin`，直接 `python3` 会缺 `yaml`）：

```bash
uv run --quiet scripts/catalog/validate_catalog_metadata.py --require-complete   # 要 0 error
uv run --quiet scripts/i18n/validate-i18n.py
```

输出末尾 `agents=N` 那个计数是关键信号——条目被判非法会从计数里消失。**注意 validate 会刷上百条 WARN**，真正的 ERROR 淹在里面，用 `grep -E "^\[ERROR"` 抓。

## 5. 硬性规则

### 5.1 ⚠️ `docs/` 当前没有运行时消费入口

这是本仓库**最大的已知缺口**。13 篇业务域文档放在仓库根目录 `docs/`，而：

- 市场详情页只渲染 `USAGE.md`，**不读 `docs/`**
- Agent 的上下文只挂载 `persona.md` / `principles.md` / `memory/` / `skills/`，**不含 `docs/`**
- 安装时整树复制会把 `docs/` 落到 `agents/<id>/docs/`（`EXCLUDED_SEGMENTS` 只排除 `upstream` / `.git` / `node_modules` / `.cache`），但**没有任何东西引导 Agent 去读它**
- `principles.md` 里引导的是各 `lark-*` 技能自己的 `SKILL.md` 与 `references/`（那是市场合集技能的目录），不是本仓库的 `docs/`

结果：这 13 篇只有访问 GitHub 的人能看到，Agent 运行时读不到。**README 里那 16 处 `docs/` 链接同样只对人有效。**

出路有两条，动手前先想清楚选哪条：

1. **本仓库自带一个私有技能**（如 `feishu-guide`），把 13 篇挪进它的 `references/`，并在 `SKILL.md` 正文里写显式索引。注意：`Skill` 工具走 `loadSkillContent` 直接返回 SKILL.md 正文，**不输出 `<skill-resources>` 清单**——光把文件丢进 `references/` 目录 Agent 一篇都看不见，必须在正文里逐行列出路径
2. **在 `principles.md` 里显式写出 `docs/` 的路径与索引**，让 Agent 知道去读

第 1 条与 `dingtalk-workspace` 的做法一致（它的 `dingtalk-guide/SKILL.md` 正文里有 13 行显式索引），代价是本仓库要新增 `skills/`。

### 5.2 `agent.json` 是 pointer 形态，schema 直接校验

inline 条目有 `projectMarketAgentConfigToAgentFs` 投影层兜底，**pointer 没有**——`agent.json` 由 `agentConfigSchema` 直接校验，根级 `additionalProperties: false`。

因此**禁止**出现这些市场展示字段：`category` / `tags` / `updatedAt` / `maintainer` / `installPolicy` / `updatePolicy` / `i18n` / `persona` / `changelog` / `contentSource`。它们的归属是 `entry.json` 与 `catalog-metadata.v1.json`。

另外两条 schema 硬要求：

- `id` 必须是 **UUID v4**（本仓库是 `9a9b9e4d-803e-43ee-a67e-9754ea6eaeab`）。`feishu-orchestrator` 这类 slug 是市场条目标识，住在 `entry.json#id`
- `avatar` 只能是 `{char, color}`（可选 `image`）。`{t, bg}` 是市场卡片字段，会被拒

**改完 `agent.json` 必须实跑校验，不能靠肉眼**：

```ts
// 在 DesireCore 主仓建一个临时 *.test.ts（放 /tmp 解析不到 @desirecore/schemas）
import { validateAgentConfig } from '@desirecore/schemas'
const r = validateAgentConfig(JSON.parse(readFileSync('<path>/agent.json', 'utf8')))
expect((r as { success?: boolean }).success).toBe(true)
```

实测这一步抓到过两轮问题：先是 8 个 `additionalProperties`，剥干净后又暴露 `id` 要 UUID、`avatar.t` 非法。

### 5.3 `USAGE.md` 只有无后缀版会被抓到

市场详情页按约定文件名取使用说明，**pointer 形态下只有无后缀 `USAGE.md` 会被远程抓取**：

1. `fetchAgentRepoData` 固定抓 `agent.json` / `persona.md` / `CHANGELOG.md` / `USAGE.md` 四个文件
2. `USAGE.<locale>.md` 的探测列表由 `extractDeclaredLocales` 从**上游 agent.json 的 `i18n.locales`** 得出——而 §5.2 说了 pointer 的 agent.json 不能有 `i18n`
3. `readAgentDetailFromPointer` 调 `resolveUsageDocFromFiles(repoData.usage, locale)` 时**不传 localeHints**，回退链只剩 `locale → 无后缀`

所以别费劲写 `USAGE.zh-CN.md`——抓不到。要双语得先修平台侧（给 pointer 路径传 locale hints）。

`USAGE.md` 有 **16000 字符上限**，超出截断到行边界。它是「装之前该知道什么」，不是全量文档。

### 5.4 平台功能说明必须逐字引用平台文案

写审批模式、路由模式这类平台功能的说明时，**说明栏必须逐字引用 `app/data/i18n/zh-CN.ts` 的 `approvalModes` 里的 `description`，不得转述**。

这条是踩过坑写下的：按模式 ID 字面反推含义会写反。实测 `ask-external` 的真实语义是「内置工具与命令放行，仅 MCP、HTTP、脚本等外部工具需审批」，而按字面推成「交给外部系统裁决」**含义相反**——对一个每次调用都在执行 `lark-cli` 的 Agent，用户照错文案选会以为收紧了审批、实际把命令执行全放行。

另外审批模式共 **7 种**（`ai-approve` / `ai-auto` / `ai-assist` / `ask-always` / `allow-listed` / `ask-external` / `allow-all`），别漏 `allow-all`。

### 5.5 公开信息边界（本仓库是公开的）

**禁止任何真实身份进入 tracked 文件**：姓名、手机号、邮箱、部门、群名、租户 ID、open_id、组织名、文档标题、个人 HOME 路径。示例统一用占位符。

真机测试跑的是真实飞书数据，任何返回业务内容的截图都会带 PII。`docs/images/` 现有两张截图是市场页面截图，不含业务数据；新增截图必须逐张人工看过。

推之前扫一遍：

```bash
grep -rInE "@[a-z0-9.-]+\.[a-z]{2,}|/Users/[a-z]+|open_id=|tenant_key=" . --exclude-dir=.git
```

### 5.6 不要把本机环境固化进公开分发物

`agent.json` 的 `llm` 保持 `smart` + `flagship` 默认，**不要**钉死到某个具体 Provider 的模型——那是本机绕配额的临时改法，别人机器上未必有那个 Provider。

## 6. 能力覆盖与已知缺口

### 与官方 CLI 的对照结论

`lark-cli` 有 23 个 Lark domain，本 Agent 的 23 个技能覆盖 22 个：

| domain | 覆盖它的技能 |
| --- | --- |
| `vc` + `minutes` + `note` | `lark-meeting` 一个技能覆盖三个 domain |
| `mindnotes` | `lark-doc`（其描述含「以及操作思维笔记」） |
| `api` + `schema` | `lark-openapi-explorer` |
| `auth` / `config` / `profile` / `doctor` | `lark-shared` |
| 其余 | 同名技能一一对应 |

**唯一没有技能覆盖的是 `application` domain**（4 条命令，都是给当前绑定的开放平台应用注册 / 管理斜杠命令）。它是开发者自管理能力，落在本 Agent 的产品定位之外，属于有意不覆盖。整个 28 技能合集里也没有覆盖它的技能。

### `principles.md` 没教 raw HTTP 逃生舱

`principles.md` 教了 `lark-cli schema <service>.<resource>.<method>` 查参数与风险，但**没教 `lark-cli api <method> <path>`** —— 官方帮助把它列为首要 Agent 工具（`Raw HTTP escape hatch — call any endpoint by path`）。512 个 shortcut 覆盖不到的长尾场景（含上面的 `application`）本来都能靠它兜住。

补这条时注意 `principles.md` 里已有的警告：静态高风险清单只覆盖 shortcut，**原生 API 层的风险不在清单里，`exit 10` 才是唯一可靠的真相源**。

### 未验证边界

`USAGE.md` 的「已知边界」表列了 8 项（会中实时内容、机器人入会、妙搭、幻灯片版式、画板编辑、实时事件、多维表格高级权限、邮件真实发送），来自 2026-09-01 的真机验证（220 个已授权 scope）。改这张表前先复核，别把「没测」写成「不支持」。

## 7. 设计决策与理由

### 为什么 `USAGE.md` 只放「装之前该知道的」

它是市场详情页渲染的内容，16000 字符上限。长文档的出路是技能 `references/`（见 §5.1 的两条出路），不是把全部塞进市场详情。`fullDesc` / `changelog` 都不该承担使用说明——`fullDesc` 来自 `persona.md`，**要进 LLM 上下文并每轮记账**，塞进去等于让用户为每轮对话付一遍安装文档的钱。

### 为什么授权那步不在同一轮里等

`principles.md` 规定用 `--no-wait` 取到链接后把控制权交还用户，等用户回复「已授权」再继续。**同一轮里先打印链接再阻塞轮询，链接根本到不了用户眼前**，最后只会超时。

### 为什么重要纪律要放进 `Must Do` 编号列表

同一条规则写在 L2 说明段里往往无效，提升为 L1 `Must Do` 的编号祈使规则才生效。改 `principles.md` 时重要纪律一律进编号列表，别写成段落。

## 8. 踩过的坑

1. **`~/.desirecore/market/official/` 是可能过期的本机副本。** 用它核对合集，实测得出过「Agent 声明了合集里不存在的 `lark-meeting`」这种错误结论——远端 `origin/main` 上合集有 28 个技能且包含它，本机副本停在只有 27 个的旧 commit。核对市场事实一律以远端为准。

2. **安装是整树复制。** `EXCLUDED_SEGMENTS` 只有 `upstream` / `.git` / `node_modules` / `.cache`，所以 `README.md` / `LICENSE` / `NOTICE` / `docs/` 全都会进用户的 AgentFS 并计入 `contentDigest`。加文件前想一下它是否该出现在用户机器上。

3. **改文件前先搜在途 PR。** 本仓库与 market 条目由多个会话并行维护，实测多次出现「刚推完才发现别人在改同一处」。开工前跑：
   ```bash
   gh pr list --repo desirecore/market --state all --search "feishu" --limit 10
   ```

4. **`validate` 的 ERROR 淹在上百条 WARN 里。** 直接看日志尾部会以为只是一堆警告，实际末尾那行 `N error(s), M warning(s)` 才是判据，用 `grep -E "^\[ERROR"` 抓具体条目。

## 9. 相关记录

| 类型 | 位置 |
| --- | --- |
| 市场卡片 | `desirecore/market` → `agents/feishu-orchestrator/` |
| 技能来源合集 | `desirecore/market` → `skills/larksuite-cli/`（pointer 到 `larksuite/cli`，28 个子技能） |
| `USAGE.md` 约定与市场详情区块 | DesireCore PR #2600 |
| `required_skills.collections` 机制 | DesireCore PR #2587（`requiredClientVersion` 因此 ≥ 10.0.144） |
| 内联市场 Agent 的 Schema 冲突修复 | DesireCore PR #2515 |
| 首次上架 | market PR #114 |
| 补 `USAGE.md` 并重新 pin | market PR #137 |

## 10. 改动前的自检清单

- [ ] 改的是本仓库还是市场卡片？（内容 → 本仓库；发版 → 卡片）
- [ ] 先搜过在途 PR，确认没人在改同一处
- [ ] `agent.json` 没混入市场展示字段，`id` 仍是 UUID，`avatar` 是 `{char, color}`
- [ ] 改了 `agent.json` 就实跑 `validateAgentConfig`，拿到 `success: true`
- [ ] 改了 `required_skills` 就同步改 `default_enabled.skills`
- [ ] 平台功能说明逐字引用 i18n 文案，没有转述
- [ ] `USAGE.md` 没超 16000 字符，没写 locale 变体
- [ ] 公开信息边界扫描零命中
- [ ] 新增 `docs/` 文档时，想清楚 §5.1 的消费入口问题
- [ ] 若要发版：先推本仓库确认 SHA 可达，再改市场三处 ref，`grep -oE '[0-9a-f]{40}'` 自检为 3 处同值
- [ ] 市场 PR 跑过 validate，`grep -E "^\[ERROR"` 零命中，`agents=` 计数没掉
- [ ] `AGENTS.md` 与 `CLAUDE.md` 内容一致
