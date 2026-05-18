---
title: AI 信息追踪 · 源清单 v1
purpose: 长期跟踪 AI 政治/经济/技术三轴向的源池，按可信度分 tier
maintainer: 你
last_updated: 2026-05
---

> **使用说明**
> 这份清单按 tier 排，每月增量更新。新源加进来要附 tier、首选 RSS / 替代抓取方式、为什么收。半年没更新或质量降级的，移到 attic 区。
> 不是 follow 越多越好——T1+T2 总数控制在 60-80 个，再多人脑顶不住。

---

## Tier 定义复习

| Tier | 性质 | 使用方式 |
|---|---|---|
| **T1 一手** | 原始事实/官方声明/学术发布 | **作为锚点**。任何结论必须能溯源到 T1。 |
| **T2 高质量分析** | 二次但增量信息（业内分析师、研究博客） | **理解上下文**。可作为"辅证"出现在摘要里。 |
| **T3 快讯聚合** | 速度优先（社交、聚合站） | **早期信号**。必须 T1 验证后才能升级 confirmed。 |
| **T4 噪音** | 二手翻译/营销稿/SEO 站 | **基本不进 pipeline**。最多用作"市场情绪"对照。 |

> 政治经济角度容易被污染（参考 Pravda 网络案例）。**默认排除**已知国家级影响力运作站点（RT/Sputnik/TASS/Pravda 网络下属域名）。

---

## T1 一手源

### A. 公司 / 实验室博客

#### 美国一线
| 名字 | 站点 | RSS | 备注 |
|---|---|---|---|
| OpenAI | openai.com/news/ | ✓ | 产品发布、研究、政策都发这里 |
| Anthropic | anthropic.com/news | ✓ | 模型发布、Claude 更新、policy |
| Google DeepMind | deepmind.google/discover/blog/ | ✓ | Gemini、AlphaFold 等 |
| Google Research | research.google/blog/ | ✓ | 偏研究 |
| Meta AI | ai.meta.com/blog/ | ✓ | Llama、FAIR |
| Microsoft Research | microsoft.com/en-us/research/blog/ | ✓ | |
| Apple ML Research | machinelearning.apple.com/research | ✓ | 罕见但重要 |
| Mistral | mistral.ai/news | 部分 | 欧洲一线 |
| xAI | x.ai/blog | 部分 | Grok、Colossus |
| Cohere | cohere.com/blog | ✓ | 企业向 |
| Stability AI | stability.ai/news | 部分 | 图像/视频 |

#### 中国
| 名字 | 站点 | RSS | 备注 |
|---|---|---|---|
| DeepSeek | deepseek.com | RSSHub | V/R 系列 |
| 阿里通义 / Qwen (research) | qwen.ai/research | 部分（JS 渲染，可能要 RSSHub 或定时爬）| 论文/技术报告主索引页 |
| 阿里通义 / Qwen (blog) | qwenlm.github.io | ✓ | 模型发布解读、技术博客 |
| 智谱 GLM | zhipuai.cn | RSSHub | |
| 月之暗面 / Kimi | moonshot.cn | RSSHub | |
| 百川智能 | baichuan-ai.com | RSSHub | |
| 字节豆包 / Doubao | doubao.com | RSSHub | |
| 阶跃星辰 | stepfun.com | RSSHub | |

> **注意**：中国实验室的发布常先在微信公众号、再在英文 GitHub README、最后才在官网。订阅时三处都要覆盖（用 RSSHub 把公众号转 RSS）。

### B. 论文 / 预印本

| 源 | URL | RSS |
|---|---|---|
| arXiv cs.CL | arxiv.org/list/cs.CL/recent | ✓ 自带 |
| arXiv cs.LG | arxiv.org/list/cs.LG/recent | ✓ |
| arXiv cs.AI | arxiv.org/list/cs.AI/recent | ✓ |
| arXiv cs.CV | arxiv.org/list/cs.CV/recent | ✓ 多模态用 |
| Hugging Face Daily Papers | huggingface.co/papers | RSSHub | 已被人工筛过的当日热门论文 |
| Papers with Code | paperswithcode.com | ✓ | 含代码 |
| ArXiv Sanity Lite | arxiv-sanity-lite.com | ✓ | 个人化筛选 |

> arXiv 噪音多，不要全订阅。**只订阅当日 listing**，配一个 LLM 摘要器把 100+ 条压成 5-10 条相关的。

### C. 模型 / 数据 / 代码发布

| 源 | URL | 抓法 |
|---|---|---|
| Hugging Face new models | huggingface.co/models?sort=created | API |
| Hugging Face trending | huggingface.co/models?sort=trending | API |
| Hugging Face datasets | huggingface.co/datasets?sort=created | API |
| GitHub trending (AI) | github.com/trending?since=daily | API/RSS |
| Ollama library | ollama.com/library | 爬虫 |

### D. 政策 / 监管 / 标准

#### 美国
- **White House AI 公告**: whitehouse.gov（搜 "artificial intelligence"）
- **NIST AI Risk Management Framework**: nist.gov/itl/ai-risk-management-framework
- **US AI Safety Institute**: nist.gov/aisi
- **FTC AI 行动**: ftc.gov（AI 相关执法）
- **联邦立法跟踪**: congress.gov（搜"AI"或具体法案号）

#### 欧盟
- **EU AI Office**: digital-strategy.ec.europa.eu/en/policies/ai-office
- **EU AI Act 实施跟踪**: artificialintelligenceact.eu（民间但很完整）

#### 英国
- **UK AI Safety Institute**: aisi.gov.uk

#### 中国
- **网信办 (CAC)**: cac.gov.cn（生成式 AI 备案、深度合成等规章）
- **中国信通院 CAICT**: caict.ac.cn（白皮书、测评报告）
- **科技部**: most.gov.cn

#### 国际
- **OECD AI Policy Observatory**: oecd.ai
- **ISO/IEC JTC 1/SC 42**: 标准制定，进度可在 iso.org 查
- **AI Index (Stanford HAI)**: aiindex.stanford.edu（年报）

### E. 财务 / 监管文件

| 源 | URL | 用法 |
|---|---|---|
| SEC EDGAR | sec.gov/edgar/search/ | 按公司搜 8-K（重大事件）、10-K（年报）、S-1（IPO）|
| 公司 IR 页 | 各公司 investor relations | 季报 + earnings call |

> AI 上市公司列表（每季度更新一次）：NVIDIA、Microsoft、Google/Alphabet、Meta、Apple、Amazon、Palantir、C3.ai、SoundHound、AMD、Intel、TSMC（ADR）、Broadcom、Arista。私营但已申报 S-1 的（如 OpenAI 若启动 IPO）必须加入。

---

## T2 高质量分析

### 英文 newsletter / 博客

| 名字 | 作者 | 频率 | 重点 |
|---|---|---|---|
| **Stratechery** | Ben Thompson | 周 4 | 战略 / 商业模式 / 平台经济 |
| **SemiAnalysis** | Dylan Patel | 不定 | 半导体 + 模型经济 + 推理成本 |
| **The Information** | 团队 | 日 | 行业内幕（付费） |
| **Import AI** | Jack Clark | 周 | Anthropic 公关之外的本人观察 |
| **Last Week in AI** | Andrey Kurenkov 等 | 周 | 综合周报 |
| **Ahead of AI** | Sebastian Raschka | 不定 | 技术深度 / 训练方法 |
| **Interconnects** | Nathan Lambert | 不定 | 后训练 / RL / 开源模型 |
| **The Gradient** | 团队 | 不定 | 学术风格深度文 |
| **Latent Space** | swyx | 周 | 工程师视角 + 播客 |
| **AI Snake Oil** | Narayanan & Kapoor | 周 | 怀疑论 / 反炒作 |
| **One Useful Thing** | Ethan Mollick | 周 | 应用 / 教育视角 |
| **ChinAI** | Jeff Ding | 周 | 中文 AI 圈英文翻译（重要！）|
| **Don't Worry About the Vase** | Zvi Mowshowitz | 周 | 综合 + 安全侧 |

### 智库 / 政策分析（政治经济专题）

| 机构 | 重点 |
|---|---|
| **Brookings AI** (brookings.edu/topic/artificial-intelligence/) | 政策分析 |
| **CSIS Strategic Tech** (csis.org) | 中美科技竞争 |
| **CSET (Georgetown)** (cset.georgetown.edu) | 安全 / 政策 / 数据 |
| **RAND AI** (rand.org) | 系统性分析 |
| **Carnegie Endowment** (carnegieendowment.org) | 国际治理 |
| **Stanford HAI** (hai.stanford.edu) | 学术界综合 |
| **Centre for the Governance of AI (GovAI)** | 牛津，长期治理 |

### 中文深度（少而精）

| 名字 | 评价 |
|---|---|
| 机器之心 jiqizhixin.com | 中文圈最好的之一，但有时是翻译稿 |
| 量子位 qbitai.com | 同上 |
| 智源 BAAI baai.ac.cn | 学术性强 |
| 少数派 sspai.com（AI 板块）| 偶有好文 |

> ⚠️ 大多数中文 AI 公众号是 T3-T4。机器之心和量子位的"翻译/转载"按转的源 tier 算，不按它们自己 tier 算。

---

## T3 快讯聚合

### X 列表（建议按主题分多个 list）

#### List 1: Researchers
Andrej Karpathy、Yann LeCun、Demis Hassabis、Jeff Dean、Ilya Sutskever、François Chollet、Sebastian Raschka、Jeremy Howard、Tri Dao、Hyung Won Chung、Noam Brown、Quoc Le

#### List 2: Lab insiders / 政策
Sam Altman、Dario Amodei、Jack Clark、Mira Murati、Greg Brockman、Mustafa Suleyman、Mike Krieger、Liam Fedus、Aleksander Mądry

#### List 3: Analysts / Journalists
Dylan Patel (@dylan522p)、swyx (@swyx)、Nathan Lambert (@natolambert)、Ethan Mollick (@emollick)、Casey Newton (@caseynewton)、Karen Hao、Will Knight、Cade Metz、Kevin Roose

#### List 4: 中国 AI 圈（X 上能找到的）
Jeff Ding (@jjding99)、Matt Sheehan、Helen Toner、Zvi Mowshowitz、Lennart Heim

> 名单会过期。建议每季度让 LLM 帮你检索"过去 6 个月在 AI 议题被高互动引用最多的 X 账号"，更新 list。

### 论坛 / 聚合

| 源 | URL | 用法 |
|---|---|---|
| Hacker News (front + ask) | news.ycombinator.com | 自带 RSS。筛 score>200 + AI tag |
| r/LocalLLaMA | reddit.com/r/LocalLLaMA | 开源模型情报 |
| r/MachineLearning | reddit.com/r/MachineLearning | 学术风 |
| r/singularity | reddit.com/r/singularity | 噪音多但偶有早讯 |
| Lobste.rs | lobste.rs | HN 替代，筛 ai tag |

### 新闻聚合 newsletter（英文）

| 名字 | 频率 | 评价 |
|---|---|---|
| **AlphaSignal** | 日 | 短小精悍 |
| **TLDR AI** | 日 | 标题党，但能扫到信息 |
| **AI News** (smol.ai) | 日 | 每日 X / Reddit / Discord 综述，质量高 |
| **Ben's Bites** | 日 | 偏产品 |
| **The Rundown** | 日 | 偏入门 |
| **The Batch** (Andrew Ng) | 周 | 教育向 |
| **DeepLearning.AI** | 周 | 同上 |

### 中文快讯（谨慎使用）

- 36氪 AI 频道 (T3-T4 边缘，看作者，不看站)
- AI 公众号矩阵（机器之心、量子位、新智元、AI前线）—— 用 RSSHub 转 RSS

---

## T4（默认不收，只在做"市场情绪"维度时取样）

- 大量中文 AI 营销公众号（"震惊！XX 颠覆 YY"风格）
- AI 行业自媒体（多为翻译稿）
- 营销博客（带 SEO 关键词堆叠）

> 这一层不进 pipeline。如果某天你要做"市场情绪"指标，可以临时采样 50-100 条标题做关键词频次分析，**但不能进可信度链路**。

---

## 政治经济角度的"对抗采样"清单

为了识别有偏向叙事，**保留** 5-10 个已知有立场的源作为对照（不进 confirmed 链，但可标记为 "biased-narrative" 出现在比较视图里）：

- 美方鹰派智库：CSIS、CNAS（Center for a New American Security）
- 美方建制：Brookings、Council on Foreign Relations
- 中国官方：人民日报英文版、新华社、China Daily、环球时报
- 行业游说：Chamber of Commerce 报告、ITI Council、中国信通院（既是 T1 又有立场）

每周自动跑一次"同一事件，五种叙事"对比，作为政治维度的"风向标指标"。

---

## ❌ 默认排除

确认有国家级影响力运作或大规模 SEO 投毒的站点，pipeline 拒绝抓取：

- RT (rt.com)、Sputnik (sputnikglobe.com)、TASS (tass.com)
- Pravda 网络下属域名（参考 NewsGuard 公开列表，约 150+ 域名）
- 已知洗稿农场（识别：发文量 vs 真实读者比例严重失衡 + 关键词堆叠 + 内容互相镜像）

---

## RSS 抓取说明

| 源类型 | 自带 RSS | RSSHub 路由 |
|---|---|---|
| arXiv | ✓ | — |
| Substack newsletter | ✓ | — |
| WordPress / Ghost 博客 | ✓ | — |
| 微信公众号 | ✗ | `/wechat/officialaccount/{biz}` |
| X / Twitter | ✗（已无官方 RSS） | `/twitter/user/{id}` 或 Nitter |
| Hacker News | ✓ | — |
| GitHub | ✓ (releases / commits / trending) | — |
| Reddit | ✓ (URL + .json 或 .rss) | — |
| Hugging Face | ✗ | `/huggingface/models` 等 |
| 新闻站 | 多数 ✓ | 没有的有 RSSHub 路由 |

> **RSSHub** 是开源的把"没有 RSS 的站"转成 RSS 的服务（github.com/DIYgod/RSSHub）。自托管或用公共实例。这个工具对中文圈几乎是必需。

---

## 实体归一表（一开始就建一份）

每抓一条新内容，LLM extract 出来的实体对照这张表归一。表也是 LLM + 人工持续维护。

```yaml
people:
  sam_altman:
    aliases: [Sam Altman, 山姆·奥特曼, 山姆奥特曼, "@sama"]
    org: openai
    role: CEO

  dario_amodei:
    aliases: [Dario Amodei, 达里奥·阿莫迪]
    org: anthropic
    role: CEO

  # ... 加到几十人的级别

orgs:
  openai:
    aliases: [OpenAI, OAI, "Open AI"]
    type: lab
    country: US

  anthropic:
    aliases: [Anthropic, ANT]
    type: lab
    country: US

  google_deepmind:
    aliases: [Google DeepMind, GDM, DeepMind, Google AI]
    type: lab
    country: US

  meta_ai:
    aliases: [Meta AI, FAIR, Facebook AI Research]
    type: lab
    country: US

models:
  gpt_5:
    aliases: [GPT-5, gpt5]
    family: gpt
    org: openai

  claude_4_x:
    aliases: [Claude 4, Claude Opus 4, Claude Sonnet 4, Claude Haiku 4]
    family: claude
    org: anthropic

  # ... 主流模型族
```

---

## 调度建议（cron 节奏）

| 源 tier / 类型 | 节奏 |
|---|---|
| T1 公司博客 | 每 1 小时 |
| T1 arXiv listing | 每天 1 次（北京时间早 9 点）|
| T1 SEC 8-K | 实时（webhook 优先），10-K/Q 每季度 |
| T1 政策站 | 每天 1 次 |
| T2 newsletter | 每天 1 次（多数周更，频抓没意义）|
| T3 X | 每 30 分钟 |
| T3 HN / Reddit | 每小时 |
| T3 daily digest（AlphaSignal 等）| 每天 1 次 |
| 实体表更新 | 手动 + 每周 LLM 建议增量 |
| 源池 review | 每月 1 次 |

---

## 输出形式（向后倒推决定 pipeline）

| 形式 | 内容 | 节奏 |
|---|---|---|
| **每日简报** | T1 confirmed + 高分 T2 解读，按"政治/经济/技术"三轴分块 | 每日早 8 点 |
| **每周深度** | 跟踪人物/公司/技术/政策的变化曲线，趋势识别 | 周日晚 |
| **告警** | T1 多源命中 + 关键词（如 "ban", "executive order", "earnings beat", "model release"）| 实时 |
| **月度回顾** | 把过去 30 天 mention 频次最高的实体拉出来做趋势报告 | 月初 |

---

## 实现栈推荐（最小可行版）

| 层 | 工具 |
|---|---|
| 调度 | GitHub Actions cron（免费）/ 自托管 cron |
| 抓取 | Python + `feedparser` + `httpx`，公众号/X 用 RSSHub |
| 去重 | SimHash + 时间窗 + LLM 兜底 |
| 抽取 | Claude Haiku / GPT-4o-mini（便宜批量）|
| 摘要 | Claude Sonnet / Opus（每日 digest 用） |
| 存储 | SQLite（< 100 万条）→ Postgres（> 100 万条）|
| 终端消费 | Obsidian vault（markdown）+ Notion db（查询）|
| 告警 | Telegram bot 或邮件 |

---

## 维护节奏

| 频率 | 动作 |
|---|---|
| 每周 | 看本周抓取量异常的源（突然 0 / 突然 10×）|
| 每月 | 跑"被引用最多但未收录"列表，决定加入 |
| 每季度 | 全量 review，移走 6 个月没产出的源到 attic |
| 每半年 | 重新评估 tier 划分（升 / 降 / 出）|

---

## attic（已退役，记录用）

（这一区一开始空着，源被移除时记到这里，包括退役原因和退役日期。半年回看是学习材料。）

---

*v1 整理完，下一步：选一两个 tier=1 源先跑通端到端 pipeline，再扩源。*
