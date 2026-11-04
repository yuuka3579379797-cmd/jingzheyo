# YoMLIA · 技术说明

> 仓库首页（介绍页）见 [`../README.md`](../README.md)。
> 仓库：https://github.com/yuuka3579379797-cmd/jingzheyo
> 提示词与运行时数据分离：仓库只放提示词、脚本和合成示例；检索结果与报告中间产物只落本地 `data/`，不进仓库、不上传。

医学与生命科学领域主题式文献追踪专家：用 **Europe PMC 检索式 + AI 精读** 把一周的增量文献压成一份可核对、可追溯的中文周报。

## 类型

Agent 型（单个 AI 专家）

## 方法：两步主干 + 两道校验

| 步骤 | 干什么 | 谁来做 | 特点 |
|---|---|---|---|
| Europe PMC 检索式 | 拆概念块 → 块内 OR、块间 AND → 日期写进每个分支 → `FIRST_PDATE` 限定 → 分项对账 | 确定性脚本 | 跑全量、不抽样、每一笔可对账 |
| AI 精读摘要 | 按「改变临床决策的可能性」取 top10–15 → 抽设计/样本量/终点/效应值+CI+P/注册号 → 给一句判断 | 模型 | 只读值得读的，不做全文精读、不做整篇翻译 |
| **哈希锚定（校验一）** | 每个关键数字附「逐字摘录 + SHA-256 短哈希」，交付前机检整份报告（不只是比对清单） | `bin/ledger.py check` | 改数字、拼句、锚定表外的裸数字都会被点名；退出码非 0 不许交付 |
| **反事实核查（校验二）** | 用本周核心结论反拼窄检索式（独立 ≥5 年窗，不套本周 7 天窗），主动找结论相左的高质量研究 | 模型 + 脚本 | 不只证明「查过了」，还证明「查过可能推翻自己的东西」 |

窗口规则：**周报窗口写显式日历日，不用 `NOW-N`DAYS**——`NOW-7DAYS` 是滚动 7×24 小时窗，会把起始日切掉大半，同一条式子不同时刻跑出不同结果。用 `bin/window.py dates` 算窗口，`bin/window.py verify` 可以现场证明两者不等价。

一句话：**把「筛」交给脚本，把「判」交给模型，把「有没有被自己骗」交给校验。**

## 为什么只适用于生物医学与生命科学

不是能力偏好，是三条硬约束：

1. **数据源**：检索走 Europe PMC（PubMed + PMC + 预印本 + 部分专利与 Agricola），收录的是生物医学文献，其他学科要么没被收录，要么题录残缺。
2. **受控词表与题录结构**：生物医学有 NLM 的 MeSH 受控词表，能把「心衰」的几十种写法归一（但注意：最近入库文献的 `meshHeadingList` 缺失率很高，2026-11-03 抽样近 7 天 50 条缺 29 条（约 58%），所以检索式不能只靠 MeSH 主干，要自由词为主、MeSH 为辅）。物理、化学、材料、工程、社科没有等价词表，也没有统一的摘要结构，只能靠自由词硬拼。
3. **精读前提**：判断「哪篇值得读」依赖 IMRaD 结构化摘要（样本量、终点、效应值、CI、P 值、注册号）。换到别的学科，摘要里没有这些字段，精读层直接失效。

**已知缺口**：中文期刊和多数未收录的会议摘要集不在覆盖范围内。方向以中文文献为主时，专家会先说明这个缺口，不硬跑。

## 用了能得到什么

- **领域最新文献自动追踪**：一次覆盖该主题这一周的**全部**增量，不是挑几篇翻译；漏了哪一类、为什么漏，账上都写着。
- **中文周报**：一句话结论 + 账目表（检索式原样、总命中、六类计数）+ top10–15 篇精读 + 综述/勘误/预印本分区 + **反向证据核查** + 下周建议检索式 + 附录（锚定表 + 全量清单）。
- **可溯源到原句**：每条结论挂 PMID/DOI，每个关键数字挂「原文摘录 + 哈希」，别人拿着报告可一条条机器核验，不用信你。
- **不自欺**：不仅证明「查过了」，还证明「查过可能推翻自己的东西」；一周阳性结果扎堆时，报告会指出这可能是真趋势也可能是检索偏倚。

## 功能清单

- **检索式工程**：把研究方向拆成 2–4 个概念块（全称 / 缩写 / 同义词 / MeSH），块内 OR、块间 AND，日期写进每个分支，产出能真正跑对的 Europe PMC 检索式。
- **集合级对账**：用 `resultType=idlist&pageSize=1000` 分别拉分支 A、分支 B、并集的 ID 集合，验 `|A∪B| = |A|+|B|−|A∩B|`；差额超 `max(2, 0.5%)` 就停下查式，不硬出报告，差额存 `mismatch.json`。
- **六类优先级归类**：勘误 > 预印本 > 述评 > 综述 > 无摘要 > 研究，每条只落一类，断言六类之和 == 命中总数（`pubType` 是多标签叠加的，独立查询相加必然重复计数）；无摘要条目单列并给 DOI。
- **摘要级精读**：取 top10–15，逐篇抽设计、样本量、主要终点、效应值 + CI + P 值、注册号，给一句判断与「对你的课题」。
- **哈希锚定**：边读边锚，每个关键数字生成 `hash12`，交付前 `ledger.py check` 机检整份报告；改写引文、跨段拼句、锚定表外的裸数字一律会被抓出。
- **反事实核查**：精读后反拼窄检索式（独立 ≥5 年窗）找反向证据，周报第 3 节必须写明「式子、时间范围、命中多少、有无相左证据」。
- **口径把关**：拒绝 EDAT 混入、跨字段 OR 静默算错（实测同一天同一主题：组内只写一次窗口 313 条 vs 逐分支各带窗口 15 条）、`NOW-N`DAYS 滚动窗切掉起始日、`sort` 缺方向回假 503、MeSH 缺失漏检、通用正则吞掉 `<0.001` 等会让周报失真的陷阱。

## 使用示例

专家卡片上的三条快捷入口，也是最快的三种用法：

- **先介绍你能做什么、不能做什么、数据存在哪（本地、不上传），再告诉我怎么用你**（第一次用先点这个）
- 帮我追一下心衰领域最近一周的新文献，出一份中文周报
- 我研究 XX 方向，帮我定检索词并拼成 Europe PMC 检索式

想按自己的方向跑，直接把「方向 + 时间窗 + 收不收综述/预印本」三件事说清楚即可。

## 技术要点

- 数据源：Europe PMC REST（免 key），端点 `https://www.ebi.ac.uk/europepmc/webservices/rest/search`
- 窗口口径：只用 `FIRST_PDATE`，且**必须写显式日历日** `[2026-10-27 TO 2026-11-02]`；`[NOW-7DAYS TO NOW]` 是滚动 7×24 小时窗，会整段切掉起始日（实测起始日 4470 条只进 1 条），同一条式子早中晚跑结果不同。用 `bin/window.py dates --days 7` 生成子句
- 翻页：`cursorMark`；明细用 `resultType=core&pageSize=25`；对账用 `resultType=idlist&pageSize=1000`（接口硬上限 1000）；`sort` 必须写方向（`P_PDATE_D desc`，缺方向会回假 503）；`synonym` 实测开关无差异，记账仅为复现

### 交付闸门 `bin/ledger.py`（零依赖，纯标准库）

```bash
# 1) 为关键数字的原句生成锚定信息（PMID 不在 abstracts.json 里直接失败，不再单源兜底）
python3 bin/ledger.py make \
  --quote "empagliflozin reduced the composite endpoint (HR 0.79; 95% CI 0.69-0.90; P<0.001)" \
  --pmid 90000001 --abstracts abstracts.json   # 示例为虚构：PMID 90000001 不存在

# 2) 六类优先级归类（每条只落一类，断言六类之和 == hitCount）
python3 bin/ledger.py classify --results results.json --out classes.json --expected-total 127

# 3) 交付前机检整份报告（退出码 0 才允许交付）
python3 bin/ledger.py check --report 周报.md --abstracts abstracts.json \
        --anchors anchors.json --ledger ledger.json --classes classes.json --json
```

`check` 的十二个机检项：锚定结构、锚定与摘要一致性、报告绑定、未登记数字、无 EDAT、检索式一致性、六类加总、反证节、无摘要单列、引用编号、精读条目锚定、取数与命中数对账。任一项不过 → `exit 1`；文件/JSON 本身有问题 → `exit 2`。

与旧工具的关系：`bin/anchor.py`（`make` / `verify`）保留，负责逐条锚定与清单比对；但它不读周报，所以交付门由 `ledger.py check` 承担。`ledger.py` 额外修掉 anchor.py 的四个已知问题：跨段拼接被分段校验拦住；摘录缺失 / 空锚定表不再静默放行；U+2212 等字符在 GBK 控制台不再崩；错误 PMID 不再靠「只有一条源」兜底通过。

归一化规则：剥已知 HTML/JATS 标签 → NFKC → 引号/短横统一 → 压空白 → 转小写。报告与原文走同一函数，保证可复现。

> **可选增强（自动部署）**：`literature-evidence-mcp` 与 `tekmeria` 由包内 `bin/setup.py` 在**首次被调用时自动安装并写入配置**，见下节。
> 未加载时自动降级到 `bin/anchor.py`，不影响交付。`med-lit-mcp` 在 PyPI 上确有公开版本（0.3.0，PolyForm Noncommercial 1.0.0，非商业许可，商用需另取授权），因许可与定位重叠未纳入默认部署。

## 首次调用自动部署（`bin/setup.py` + `bin/bootstrap.sh`）

专家被调用的第一件事是自检本机有没有这两个本地 MCP 服务；缺就装，装完写配置、做握手验证。**不需要用户手动 pip install。**

```bash
bash bin/bootstrap.sh --check            # 只体检，不改任何文件
bash bin/bootstrap.sh --check --json     # 同上，机器可读（Agent 用这个）
bash bin/bootstrap.sh --install --json   # 缺什么补什么（pip + 写 mcp.json + 握手）
bash bin/bootstrap.sh --bg               # 后台静默预热（默认不写日志；要日志加 --diagnose）
```

`bootstrap.sh` 只负责挑对一个 ≥3.10 的 Python（`python3` / `python` / `py -3`），真正的活在 `setup.py` 里，也可以直接调：

| 它会做什么 | 具体行为 |
|---|---|
| 装依赖 | 用**当前解释器** `python -m pip install`，不猜别的 python，避免装到别处；已装则跳过（幂等） |
| 写配置 | 合并进 `~/.workbuddy/mcp.json`，**只增不删**，保留你原有的其它 server 和字段；写前自动备份 `mcp.json.bak-<时间戳>` |
| 损坏保护 | 现有 `mcp.json` 不是合法 JSON 时**拒绝写入**并原样报错，绝不覆盖你的配置 |
| 验证 | 给每个服务发真实 `initialize` 握手，拿到 `serverInfo` 才算成功；进程起不来就报失败，不谎报 |
| 失败时 | 原样打印 `failures`（没 pip / 离线 / Python < 3.11 / 配置损坏），专家随即降级到 `bin/anchor.py` 并在周报里声明 |

**唯一需要人工的一步**：MCP 只在会话启动时加载，脚本替不了。装完要在「连接器管理 → 右上角自定义」对两个服务点**信任**，再**新开一个对话**才生效。

> **为什么不是装包即部署**：WorkBuddy 的专家包禁止携带 `hooks/`（`validate_expert.py` 会直接报 `Forbidden directory`），拿不到 SessionStart 钩子。所以部署入口放在**主提示词的「启动自检」章节 + 包内脚本**：专家一被调用就先跑自检，等价于「调用即部署」。想彻底免提示词，只能把它做成普通插件（而非专家包）。

零依赖、纯标准库，Windows / macOS / Linux 通用。退出码：`0` 就绪、`1` 部署失败、`2` 部分可用（已降级）。关闭自动部署：`export LITRADAR_NO_AUTO_SETUP=1`。

## 反馈渠道

**本项目不提供邮箱反馈，也不收集任何用户数据。** 所有沟通留在 GitHub，便于后来的人检索到同样的答案：

| 你想做的事 | 渠道 |
|---|---|
| 使用问题、一般反馈、经验交流 | [Discussions](https://github.com/yuuka3579379797-cmd/jingzheyo/discussions) |
| 可复现的 Bug（有命令、有报错） | [Issues](https://github.com/yuuka3579379797-cmd/jingzheyo/issues) |
| 安全漏洞 | [私有漏洞报告](https://github.com/yuuka3579379797-cmd/jingzheyo/security/advisories/new) —— **不要公开** |
| 行为准则问题 | 见 `CODE_OF_CONDUCT.md` 的私密渠道 |

> 仓库地址已在 `README` / `FEEDBACK` / `CONTRIBUTING` / `SECURITY` / `CODE_OF_CONDUCT` / `config.example.env` / `prompts/feedback_guidance.md` / 主提示词八处对齐，改仓库时一起改，别只改一处。

Issue 格式（五类：A 检索式 / B 交付门误判 / C 部署失败 / D 判断有误 / E 文档与实现不一致）与证据要求见 [`FEEDBACK.md`](../FEEDBACK.md)。

## 日志与隐私

- **默认不生成任何日志文件**，不收集、不上传任何用户数据。
- 需要排查时显式开启：`bash bin/bootstrap.sh --check --diagnose`（或 `EXPERT_LOG=1`），日志写到 `data/logs/setup.log`。
- 日志写盘前做脱敏：用户主目录替换为 `[HOME]`，不记录 env 值与完整 JSON。
- 报错信息**不会**引导用户上传日志，只引导到 Discussions / Issues；是否贴日志、贴哪一段，由用户自己决定。
- 运行时产物（`abstracts.json` / `anchors.json` / `classes.json` / `ledger.json` / `mismatch.json`）只落本地 `data/` 或当前目录，已被 `.gitignore` 忽略，不会进仓库。
- 提示词库（`prompts/`、`agents/`）只有模板与 `SYNTHETIC` 合成示例，不含真实题录、用户输入或凭据——库与数据是分开的两套东西。

## 目录结构

```
.
├── agents/medical-lit-radar.md   # 系统提示词（唯一生效的那份）
├── prompts/                      # 可复用片段（feedback_guidance.md 等）
├── bin/                          # ledger.py（交付门）/ window.py（窗口）/ anchor.py / setup.py / bootstrap.sh / export_prompt.py（导出通用提示词）
├── docs/                         # deploy.md（怎么装）/ portability.md（跨模型）/ pack-readme.md（本文）
├── tests/fixtures/               # 合成测试数据（SYNTHETIC，可跑通交付门）
├── data/  logs/                  # 运行时目录，只留 .gitkeep，内容不进仓库
├── avatars/expert-v3.png         # 头像（AI 生成，随包授权）
├── config.example.env            # 环境变量示例（只放非敏感值）
├── .gitignore                    # 运行时数据/凭据/中间产物一律忽略（zip 内已含）
├── README / CONTRIBUTING / SECURITY / CODE_OF_CONDUCT / FEEDBACK / CHANGELOG / LICENSE
└── .gitignore
```

## 头像

当前头像 `avatars/expert-v3.png` 由作者用 AI 绘图生成，随本包按 CC BY-NC-SA 4.0 一并授权。如需替换，要求：
- 格式：PNG（推荐）或 JPG
- 尺寸：512×512 px
- 大小：单张不超过 500KB

## 安装：把包拖进去，让 AI 装

不需要安装程序。它是**一个文件夹**——提示词、几个 Python 脚本、一张头像。所谓安装，就是放到对的位置、让宿主认得它。

**最省事：把 `medical-lit-radar.zip` 拖进对话框**（或解压后拖文件夹、或把路径发过去），贴上这段：

```text
这是一个 WorkBuddy 专家包（显示名 YoMLIA），帮我装好，不要跳步骤：
1. 找到包根目录（有 .codebuddy-plugin/plugin.json 的那一层）。
2. 复制到 ~/.workbuddy/plugins/marketplaces/my-experts/plugins/medical-lit-radar/
   （Windows 为 C://Users//<你>//.workbuddy//plugins//marketplaces//my-experts//plugins//medical-lit-radar//）
   已存在就先备份成 medical-lit-radar.bak-<日期> 再覆盖。
3. 到 WorkBuddy 安装目录下找 resources/app.asar.unpacked/resources/plugins/
   workbuddy-builtin/skills/expert-manager/scripts/ ，先跑 validate_expert.py <专家目录>，
   通过后再跑 register_expert.py <专家目录>。
4. 把两条命令的完整输出贴给我，并告诉我还要做什么（通常是重启 WorkBuddy）。

validate 不通过就停下，把报错原文给我，不要跳过校验直接注册。
```

重启后进【专家中心 → 我的专家】，看到 **YoMLIA** 和橘发拿书的头像就成了。

**DeepSeek Harness / 千问 Code / Claude Code / Cursor** 没有专家中心，装法是另一套——不用注册，用「设提示词 + 放脚本」：

```text
用 python3 bin/export_prompt.py --out yomlia-system.md 导出系统提示词，
内容设为你的系统提示词（或 AGENTS.md / CLAUDE.md）；bin/ 保留在工作目录供后续调用。
```

各平台完整步骤、手动命令、四个常见坑，见 **[`docs/deploy.md`](deploy.md)**。

> 手动装只有三步：`cp -r` 到专家目录 → `validate_expert.py` → `register_expert.py`。两个脚本在 WorkBuddy 安装目录的 `skills/expert-manager/scripts/` 里，不在本包内。

## 接到别的 AI 里跑

这个包不是死绑在 WorkBuddy 上的一段提示词，它是**三层可以拆开用**的东西：

| 层 | 文件 | 换客户端要改什么 |
|---|---|---|
| 提示词层 | `agents/medical-lit-radar.md` 正文 + `prompts/feedback_guidance.md` | 什么都不用改，纯文本 |
| 脚本层 | `bin/*.py` | 需要 Python 3.10+ 与能执行命令的环境 |
| MCP 层（可选） | `literature-evidence` / `tekmeria` | 标准 stdio MCP，支持 MCP 的客户端都能挂 |

一句话导出成任何模型的 system prompt：

```bash
python3 bin/export_prompt.py --check                          # 体积与估算 token（约 1.3–1.9 万）
python3 bin/export_prompt.py --out yomlia-system.md           # 通用 markdown
python3 bin/export_prompt.py --format json --model <model> --out system.json   # API 直发
python3 bin/export_prompt.py --no-runtime --no-guidance --out slim.md          # 瘦身版
```

导出时会剥掉 WorkBuddy 的 frontmatter，并补一段**运行环境说明**：告诉模型自己有没有 shell、没有时只能给检索式与框架、不许编 hitCount。

客户端接法（Claude Code / Cursor / API / 网页版）、能力矩阵、换模型后最先退化的三件事，见 **[`docs/portability.md`](portability.md)**。

> 诚实提醒：**网页版聊天窗没有 shell，出不了带数字的周报**。它能帮你把检索式拼好、把判断框架讲清，数字得你在本机跑完回填。任何声称「网页版也能出完整周报」的说法，都是编的。

## 打包分享

```bash
python3 scripts/package_expert.py <expert-dir>
```

## 版本与变更（v1.5.2）

1. **打包产物补上 `.gitignore`**：`package_expert.py` 会跳过点文件，所以以前 zip 里没有 `.gitignore`、`data/.gitkeep`、`logs/.gitkeep`——传到 GitHub 等于没装锁。现在打包后自动注入这三条记录。
2. **清掉 4 处真实题录示例**：README、`bin/anchor.py`、`bin/ledger.py` 的 `--pmid 36331121` 与主提示词里的 example handle `pmid:30831578/...` 全部换成合成号 `90000001`，并注明「示例为虚构」。
3. **git 实测**：`git init` + 生成 `data/abstracts.json`、`data/expert.db`、`logs/setup.log`、`.env`、`secrets.txt` 后 `git add -A`，只有 `.gitignore` 与两个 `.gitkeep` 及包内文本进暂存区，运行时产物全被忽略。

## 版本与变更（v1.5.1）

1. **安装改成「拖给 AI 装」**：README 与 GitHub 页把「拖进对话 → 让 AI 复制、校验、注册」放在最前面，并给出可复制的部署指令；新增 `docs/deploy.md`，覆盖 WorkBuddy（专家中心）、DeepSeek Harness / 千问 Code / Claude Code / Cursor（设提示词 + 放脚本）、网页版（只能提示词）三种场景，附手动三步与四个常见坑。
2. **一致性与叙述复查（改掉 6 处）**：
   - `--bg` 的说明还写着「日志落 `~/.workbuddy/litradar/setup.log`」，与「默认不写日志」矛盾 → 改为「默认不写，要日志加 `--diagnose`」；
   - 「使用示例」还停在旧的三条 quickPrompt（第三条已被替换）→ 与实际卡片对齐；
   - `config.example.env` 里 `LITRADAR_AUTO_SETUP=1` 与脚本读取的 `LITRADAR_NO_AUTO_SETUP` 是两套命名 → 统一为后者，脚本保留旧名兼容；
   - `EXPERT_DIAGNOSE` 只在示例里出现、代码没读 → `bootstrap.sh` 现在真读它；
   - 反馈模板把版本写成固定值（`v1.4.0` / `v1.5.0`）→ 改为「照抄 plugin.json」，避免下次又滞后；
   - v1.2.0 历史条目里「`NOW-7DAYS` 等于显式 7 天窗」没标注被推翻 → 加删除线 + 说明。
3. **导出脚本的标题层级修正**：附录嵌入后 `##` 与「## 附录」同级 → 整体再降一级（`###` / `####`）。

## 版本与变更（v1.5.0）

1. **新增 `bin/export_prompt.py`**：把提示词正文剥出来、补「运行环境说明」，导出成任何客户端可用的 system prompt（`md` / `text` / `json` 三种格式，`--check` 看体积）。完整版约 2.1 万字符、估算 1.3–1.9 万 token；`--no-runtime --no-guidance` 瘦身到 1.1 万。
2. **新增 `docs/portability.md`**：三层拆解、7 类客户端的能力矩阵、Claude Code / Cursor / API / 网页版四种接法、MCP 配置片段、换模型后最先退化的三件事、新客户端自测清单。
3. **README 增「接到别的 AI 里跑」**：讲清「模型可换、标准不换」——交付标准写在脚本里，不写在模型的脾气里。
4. **诚实边界写进文档**：网页版没有 shell，只能给检索式与框架，出不了带数字的周报。

## 版本与变更（v1.4.1）

1. **引导词改口**：首条 quickPrompt / `defaultInitPrompt` 从「介绍能做什么、不能做什么」扩到「**再交代数据存在哪（本地、不上传）**」，因为提示词库与运行时数据已经分开，这件事要用户第一眼就知道。
2. **主提示词新增「数据放在哪」一节**（原「四件事」→「五件事」），并加了「报告中间产物只写 `data/`，绝不写进 `prompts/` 或 `agents/`」的硬规矩——真实题录混进提示词库，等于把用户数据带进了要公开的库。
3. **仓库地址落地**：`<OWNER>/<REPO>` 全部替换为 `yuuka3579379797-cmd/jingzheyo`（8 个文件），`homepage` 同步。
4. **头像换新**：`avatars/expert-v3.png`（512×512，377 KB，AI 生成、无第三方版权），替换旧的第三方插图；`LICENSE` 的「头像例外」条款随之删除，改为随包授权。
5. **简介微调**：45 → 47 字，补「数据只存本地」。

## 版本与变更（v1.4.0）

1. **窗口口径纠错（重要）**：上一版写「`NOW-7DAYS` 等于显式 7 天窗」，**这是错的**。2026-11-03 20:36 在糖尿病肾病主题上实测：`NOW-7DAYS` 49 条 ｜ 显式 7 日 62 条 ｜ 显式 8 日 71 条；边界日实验显示起始日 2026-10-27 全天 4470 条中只有 **1** 条落在 NOW 窗内。根因：`NOW-7DAYS` 是滚动 7×24 小时窗，边界随"此刻"移动，起始日被切掉大半，当日只进已入库的少量记录。上一版之所以测出 14=14，是因为在**凌晨 00:50** 跑的、且主题太小、10-27 当天没有记录——两个窗口恰好近乎重合，属于无效验证。新增 `bin/window.py`（`dates` 算显式窗口、`verify` 现场对比三窗并做边界日实验），提示词与文档全部改为显式日历日。
2. **改名 YoMLIA**：displayName 与提示词自称统一为 YoMLIA；技术 id 仍为 `medical-lit-radar`（改动 id 会丢失注册与市场条目）。
3. **发布前安全收口**：移除全部个人邮箱，反馈改 GitHub Discussions / Issues / 私有漏洞报告（仓库：`yuuka3579379797-cmd/jingzheyo`，v1.4.1 已填实）；日志默认关闭，改为 `--diagnose` / `EXPERT_LOG=1` 才写且落盘前脱敏；新增 `.gitignore`、`CONTRIBUTING.md`、`SECURITY.md`、`CODE_OF_CONDUCT.md`、`config.example.env`、`prompts/`、`tests/fixtures/`（合成数据，可跑通 12 项交付门）、`data/` `logs/` 占位。

## 版本与变更（v1.3.0）

v1.3.0 是把 v1.2.0 的两拨改动合并，并**逐条复跑**了其中的数字断言：

| 断言 | 复核结果 | 处理 |
|---|---|---|
| `[NOW-7DAYS TO NOW]` == 显式 7 天窗 | ⚠️ **测量无效，结论错误**（当时在凌晨 00:50、主题太小、边界日无记录，两窗恰好重合）→ v1.4.0 推翻 | 已改为显式日历日，见 v1.4.0 |
| `pageSize` 硬上限 1000 | ✅ 成立（1001 → `errCode 404: Valid size is between 1 and 1000`） | 采纳 |
| `sort` 不带方向会失败 | ✅ 成立（`P_PDATE_D` → HTTP 503，加 `desc` 正常） | 采纳 |
| 六类独立查询不能相加 | ✅ 成立，但数字不同（102 条里 JA 92 / Review 70 / 同时 64，相加 162 > 102） | 采纳结论，换成实测值 |
| MeSH 缺失率高 | ✅ 方向成立，比例不同（近 7 天抽样 50 条缺 29 条 = 58%） | 采纳结论，换成实测值 |
| 跨字段 OR 会算错 | ✅ 成立，倍数不同（313 vs 15） | 采纳结论，换成实测值 |
| `synonym` 默认 true 会差数倍 | ❌ **未复现**：缺省 / `true` / `false` 三种设置命中数完全相同（6,544 与 220,864 各自不变） | 改写成「开关无差异，别拿它解释命中数」 |
| `PUB_TYPE:"Review"` 与叠加后命中数相同 | ❌ **未复现**：70 vs 64 | 改写成实测计数，结论（不能相加）保留 |

另外更正两处**我自己**的事实错误：`med-lit-mcp` 在 PyPI 确有公开版本（0.3.0，非商业许可），此前写「未查到公开安装源」不成立；通用正则误伤率原写「61%」，2026-11-03 抽样 50 条实测为 20%（10 条，吞的是 `<0.001` 与 `<h4>`）。

## 版本与变更（v1.2.0）

v1.2.0 针对「规约承诺 vs 实际可执行」的差距做了三项改造，均以**可机检**为准，不靠自我声明：

1. **交付门落地为脚本**：新增 `bin/ledger.py check`，机检十二项（相当于把原来的「发布前自查十二项」从模型自证变成 `exit code`）。退出码非 0 不许交付；失败项以 JSON 输出，可逐条复现。并把这四类问题（跨段拼接、缺哈希静默通过、空锚定表 0/0 放行、U+2212 在 GBK 控制台崩溃）在 `ledger.py` 里堵掉——注意 `anchor.py` 本身没改，旧入口仍会放过这些情况，交付请走 `ledger.py`。
2. **日期与检索口径按实测修正**：~~`[NOW-7DAYS TO NOW]` 实测等于显式 7 天窗~~（**该结论已被 v1.4.0 推翻**：当时是凌晨 00:50、主题太小、边界日无记录导致的无效测量），删除「多给一天 / 写 `NOW-8DAYS`」的旧建议（它会让每周窗口与上期重叠一天）；对账改为 `resultType=idlist&pageSize=1000` 拉 ID 集合算 `|A∪B| = |A|+|B|−|A∩B|`；补上 `sort` 缺方向会回假 503 这个坑（`synonym` 实测开关无差异，不再作为口径风险）；明确 `MESH` 在最近入库记录里缺失率高（近 7 天抽样约 58%），只能作辅助分支。
3. **六类分流改为优先级归类**：`pubType` 是多标签叠加（2026-11-03 实测同一批 102 条：`Journal Article` 92、`Review` 70、两者同时 64；四个独立分类查询相加 162 > 总数 102），独立查询相加必然重复计数。改为「勘误 > 预印本 > 述评 > 综述 > 无摘要 > 研究」每条只落一类，并断言六类之和 == 命中总数（`bin/ledger.py classify`）。

反事实核查同步调整为**独立 ≥5 年时间窗**：能推翻本周结论的研究通常更早、更大，套用本周 7 天窗会把它结构性排除，导致「已检索，未见相左证据」成为设计保证的结论。

## 版权与许可

本专家包（提示词、流程、模板文本）采用
**[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh-hans)**
许可：可自由复制、修改、再分发，但须**署名**、**不得商用**、**修改后须以相同协议共享**。
详见 `LICENSE`。

- 作者：惊蛰yo
- 头像 `avatars/expert-v3.png` 由作者使用 AI 图像生成工具制作，随本包一同授权；若你所在司法辖区对 AI 生成物的版权归属另有规定，以当地法律为准。
- 数据源 Europe PMC 由 EMBL-EBI 提供，使用时请遵守其服务条款。
- 本专家输出文献摘要级情报，不构成临床决策建议。
