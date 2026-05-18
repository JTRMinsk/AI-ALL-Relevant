# 个人知识 SRS 系统 · 项目设计总览

> **版本**:v1.1 设计定稿  
> **日期**:2026-05-02  
> **状态**:设计阶段完成,待 Cursor 实现  
> **更新**:v1.1 — 新增第十四节"数据生命周期",细化 sync 子命令、概念/题目 ID 稳定性约定、删除语义、统计重算策略;原"风险约束"改为第十五节

---

## 一、项目目标

构建一个**个人化的间隔重复(SRS)学习系统**,把零散的学习笔记转化为可测试、可追踪、可视化的知识体系。

**核心痛点解决**:
- 笔记写完就忘——通过测试主动巩固
- 不知道自己哪里弱——通过数据沉淀客观定位
- 概念之间的关系记不清——通过图谱可视化
- 不想花时间编题——交给 LLM 自动生成

**与 Anki 等通用工具的差异**:
- **重知识结构**:不只测单点,还测概念之间的关系、概念到案例的映射
- **三态作答**:对/错/不知道(传统 SRS 只有对错)
- **图谱视图**:能看到知识地图上薄弱区的分布
- **数据驱动**:每个权重、阈值都可配置和微调

---

## 二、用户工作流

```
学习输入(课堂录音/照片/资料)
       │
       │  你 + 转录工具 + Claude
       ↓
笔记整理(Markdown 笔记 *.md)
       │
       │  Claude + 抽取 prompt(srs-extraction-prompt.md)
       ↓
知识结构化(concepts.yaml + questions.yaml)
       │
       │  loader.py 同步到数据库
       ↓
数据库就绪(srs.db)
       │
       │  cli.py review 命令
       ↓
复习答题(三态:✓ / ✗ / ?)
       │
       │  作答记录 → 调度器 → 统计聚合
       ↓
会话报告 + 交互式图谱(report.html)
       │
       │  浏览器双击打开,可分享、可归档
       ↓
基于薄弱点决定下次复习什么
       │
       └──→ 回到笔记修改 / 进入下一轮复习
```

---

## 三、整体架构

```
┌────────────────────────────────────────────────────────────┐
│ 输入层:Markdown 笔记                                        │
│ notes/*.md (你手写,不在版本控制循环里)                       │
└─────────────────────┬──────────────────────────────────────┘
                      ↓ Claude + 抽取 prompt
┌─────────────────────┴──────────────────────────────────────┐
│ 知识层:结构化 YAML(可读、可 git diff、可手改)              │
│ knowledge/                                                  │
│   ├─ concepts/*.yaml    (概念库,每篇笔记一文件)             │
│   └─ questions/*.yaml   (题库,每篇笔记一文件)               │
└─────────────────────┬──────────────────────────────────────┘
                      ↓ loader.py(增量同步)
┌─────────────────────┴──────────────────────────────────────┐
│ 数据层:SQLite 单文件                                        │
│ data/srs.db                                                 │
│   ├─ concepts            (从 yaml 同步)                      │
│   ├─ questions           (从 yaml 同步)                      │
│   ├─ review_log          (append-only,永不删除)             │
│   ├─ concept_stats       (聚合视图,定期重算)                │
│   └─ question_review_state (SRS 调度器状态)                  │
└──┬──────────────────┬──────────────────┬──────────────────┘
   ↓                  ↓                  ↓
[CLI 复习器]    [报告生成器]      [图谱生成器]
做题 + 3 态判定  会话总结         pyvis 交互图谱
   │                  │                  │
   ↓                  └──────┬───────────┘
回写 review_log              ↓
                    reports/YYYY-MM-DD-HHMM.html
                    (会话报告 + 图谱 合一)
                            │
                  ┌─────────┴──────────┐
                  │ config.yaml(横向) │
                  │ 权重、阈值、调度 │
                  └────────────────────┘
```

---

## 四、核心数据模型

### 4.1 概念(concepts.yaml)

由 Claude 从笔记抽取,可手改:

- `id`(kebab-case 唯一)、`name`、`type`(7 类:concept/parameter/tool/process/method/case/gotcha)
- `domain`(领域,如 vllm/gpu-hardware/ai-trainer)
- `source_file` + `source_section`(可溯源到原笔记小节)
- `definition`(一句话定义)+ `key_facts`(2-6 条具体事实)
- `relations`(4 种关系:compare_with / belongs_to / affects / flow_next)
- `gotchas`(⚠️ 标记的坑)、`examples`(具体案例)

### 4.2 题目(questions.yaml)

由 Claude 生成,4 种题型:

- `single_choice`(单选)
- `multi_choice`(多选)
- `matching`(匹配)
- `ordering`(排序)

每题包含:`id`、`type`、`difficulty`(1-5)、`concept_ids`(关联概念)、题干、选项/配对/序列、`explanation`(必填,3-4 行,引用原笔记)。

**强制配额**(prompt v1.1):
- 单选 ≤ 60%,其他题型最低占比有保证
- 跨概念组合题 ≥ 10%
- 难度 5 陷阱题 ≥ 10%(覆盖 ⚠️ 勘误)

### 4.3 作答记录(review_log,append-only)

| 字段 | 含义 |
|---|---|
| `timestamp` | ISO 时间戳 |
| `question_id` | 题目 ID |
| `concept_ids` | 这道题考的概念(JSON 数组) |
| `outcome` | `correct` / `wrong` / `unknown` 三态 |
| `time_spent_sec` | 答题耗时 |
| `user_note` | 选填,当场备注 |
| `question_quality` | 选填,👍/👎/🚮 题目质量反馈 |

**永不删除**——这是学习的完整轨迹。

### 4.4 概念统计(concept_stats,定期重算)

每个概念一行,关键字段:
- `total_reviews`、`correct_count` / `wrong_count` / `unknown_count`
- `accuracy`(原始正确率)
- `weighted_score`(0-1,加权后的掌握度)
- `mastery_level`(weak / shaky / stable / mastered 四级)
- `last_reviewed_at`、`next_due_at`

### 4.5 题目调度状态(question_review_state)

每道题一行,SM-2 算法用:
- `easiness`(易度因子,2.5 起步)
- `interval_days`(当前复习间隔)
- `repetitions`(连续答对计数,答错清零)
- `next_due_at`

---

## 五、核心算法

### 5.1 三态调度

```
correct  → SM-2 标准升级,间隔变长
wrong    → 重置,1 天后再考
unknown  → 比 wrong 更激进,6 小时后再考,易度因子 ×0.8
```

`unknown` 比 `wrong` 优先级更高,因为它代表知识盲区(不是记忆失误)。

### 5.2 加权掌握度公式

```
weighted_score = w1 × accuracy 
               + w2 × recency_factor 
               + w3 × consistency_factor
```

默认权重 `0.5 / 0.3 / 0.2`,但**全部可在 config.yaml 配置**。

- `accuracy` = 正确率
- `recency_factor` = 时间衰减(最近答对的权重 > 半年前答对)
- `consistency_factor` = 多次稳定答对 > 偶尔答对一次

### 5.3 四级掌握度判定

| 等级 | weighted_score | 视觉 | 调度处理 |
|---|---|---|---|
| 🔴 weak | < 0.5 | 红 | 高频出现 |
| 🟡 shaky | 0.5-0.7 | 黄 | 中频 |
| 🟢 stable | 0.7-0.9 | 绿 | 低频 |
| ⚪ mastered | > 0.9 且测过 ≥5 次 | 白 | 暂停出题 |

### 5.4 图扩散(根因诊断,可选)

利用概念关系,从弱概念向外传染:
- `compare_with` 关系上的对面概念也值得检查
- `affects` 关系上的"原因端"是潜在根因

例:KV Cache 弱 + GQA affects KV Cache → 提示"根因可能是 GQA"。

---

## 六、可配置项(config.yaml)

```yaml
weights:
  accuracy: 0.5
  recency: 0.3
  consistency: 0.2

thresholds:
  weak: 0.5
  shaky: 0.7
  stable: 0.9
  mastered_min_reviews: 5

scheduling:
  unknown_interval_hours: 6
  wrong_interval_hours: 24
  unknown_easiness_penalty: 0.8
  sm2_initial_easiness: 2.5

report:
  generate_session_report: true
  generate_weekly_report: false   # MVP 关闭,留口子
  output_dir: "./reports"
  include_graph: true
```

---

## 七、报告与可视化

### 7.1 会话报告(每次复习完自动生成)

格式:**单一 HTML 文件**,双击浏览器打开,无需服务器。

内容结构:
```
顶部:本次会话总结
  ├─ 答题统计(总数/对/错/不知道)
  ├─ 本次正确率 + 与历史对比趋势箭头
  └─ 时间消耗

中部:薄弱概念诊断
  ├─ 🔴 重点关注(本次新发现的弱概念,带解析)
  ├─ 🟡 摇摆区
  └─ 🟢 巩固区

中部:交互式概念图谱(pyvis)
  ├─ 节点 = 概念,颜色 = 掌握度
  ├─ 节点大小 = 测试次数
  ├─ 边 = 关系(不同样式区分 compare_with/affects/flow_next)
  └─ 鼠标悬停看详情,可拖拽

底部:本次错题清单
  ├─ 题目原文
  ├─ 你的回答 vs 正确答案
  └─ 解析 + 跳回原笔记的链接(source_section)
```

### 7.2 周报(MVP 不做,留命令口子)

```bash
srs report --weekly  # 暂时输出 "Not implemented"
```

代码层预留接口,后期需要时实现。

### 7.3 终端简版图谱(开发调试用)

`rich` 库在终端打印带颜色的概念列表,粗糙但能跑,纯文本。

---

## 八、技术栈

| 用途 | 工具 | 说明 |
|---|---|---|
| 语言 | Python 3.10+ | 你已有 |
| 数据校验 | Pydantic | YAML → 强类型对象 |
| YAML 解析 | PyYAML | 标准库 |
| 数据库 | sqlite3 | Python 内置 |
| CLI 框架 | Typer | 命令行交互 |
| 终端美化 | Rich | 彩色输出、表格 |
| 图算法 | NetworkX | 关系图、扩散 |
| 图渲染 | PyVis | 交互式 HTML |
| HTML 模板 | Jinja2 | 报告组装 |
| LLM 调用 | anthropic | 程序化生成题目(可选) |

**全部 pip 可装,无系统级依赖。**

---

## 九、目录结构

```
ai-srs/
├── pyproject.toml              # 项目配置 + 依赖
├── README.md                   # 项目说明
├── config.yaml                 # 用户可改配置
│
├── prompts/
│   └── srs-extraction-prompt.md  # v1.1 抽取 prompt(已完成)
│
├── notes/                       # 你的原始笔记(.gitignore 可选)
│   ├── OpenClaw部署-完整图解版.md
│   ├── GPU_维度对大模型的影响完全指南.md
│   └── ...
│
├── knowledge/                   # 结构化知识(进 git)
│   ├── concepts/
│   │   ├── openclaw.yaml
│   │   └── gpu-dimensions.yaml
│   └── questions/
│       ├── openclaw.yaml
│       └── gpu-dimensions.yaml
│
├── data/                        # 运行时数据(.gitignore)
│   └── srs.db                   # SQLite 数据库
│
├── reports/                     # 历史报告(.gitignore 或部分进 git)
│   ├── 2026-05-02-1430.html
│   └── ...
│
├── src/srs/
│   ├── __init__.py
│   ├── models.py                # Pydantic 数据模型
│   ├── config.py                # 配置加载
│   ├── loader.py                # YAML → SQLite 同步
│   ├── scheduler.py             # SM-2 + 三态调度
│   ├── stats.py                 # 加权分数 + 概念聚合
│   ├── reporter.py              # 报告生成
│   ├── graph.py                 # NetworkX 图构建 + 扩散
│   ├── visualizer.py            # PyVis HTML 生成
│   ├── cli.py                   # Typer CLI 入口
│   └── templates/
│       └── report.html.j2       # Jinja2 模板
│
└── tests/                       # 测试(后期补)
    └── ...
```

---

## 十、CLI 命令规划

```bash
# 同步知识(YAML → DB)
srs sync                    # 默认:upsert,不删除任何数据库记录
srs sync --dry-run          # 预演,只报告会改什么,不真改
srs sync --prune            # upsert + 清理"YAML 里已删除"的孤儿
srs sync --strict-rename    # 概念定义有重大变化时停下来询问
srs sync --concepts-only    # 只同步概念,不动题目
srs sync --rebuild-stats    # 同步后强制全量重算 concept_stats

# 复习
srs review                  # 开始默认复习会话(SRS 调度推荐题目)
srs review --concept <id>   # 专项复习某个概念
srs review --domain vllm    # 专项复习某个领域
srs review --count 20       # 限定题目数量

# 报告与图谱
srs report                  # 生成最近一次会话报告
srs report --weekly         # MVP 占位,提示未实现
srs graph                   # 单独生成全局知识图谱(不含会话数据)

# 工具
srs status                  # 看 YAML/DB 同步状态、待 upsert 数量、待 prune 孤儿数
srs stats                   # 终端打印概念掌握度统计
srs config                  # 显示当前配置
srs init                    # 初始化项目(建库、装样例)
```

---

## 十一、实施路线图

### MVP(最小可用版本)

按依赖顺序逐步实现:

1. **项目骨架**:目录结构、pyproject.toml、config.yaml 模板、空模块文件 + docstring
2. **数据模型**:Pydantic 类(Concept/Question/ReviewLog/...)+ SQL 建表脚本
3. **loader**:读 YAML 写 SQLite,增量同步逻辑
4. **scheduler**:SM-2 + 三态规则,~80 行
5. **stats**:加权分数计算,概念级聚合
6. **CLI 复习核心**:`srs review` 命令,终端答题循环
7. **reporter + visualizer**:Jinja2 模板 + PyVis 集成,生成 HTML 报告
8. **graph**:全局图谱视图(`srs graph` 命令)

### 后续(MVP 跑顺后再加)

- 周报(`srs report --weekly` 实现)
- Streamlit web app(图谱→直接进入复习)
- 题目质量反馈机制(👍/👎/🚮 累积淘汰烂题)
- 历史趋势折线图
- 导入/导出(便于换设备、备份)

---

## 十二、已确认的设计决策

| # | 决策点 | 结论 |
|---|---|---|
| 1 | 题型 | 单选 + 多选 + 匹配 + 排序,无开放题 |
| 2 | 题目来源 | 全部 LLM 生成,你不写题 |
| 3 | 笔记输入 | 你继续写,Claude 抽取 |
| 4 | 数据格式 | 内容用 YAML(diff 友好),状态用 SQLite |
| 5 | 三态作答 | correct / wrong / unknown,unknown 优先级最高 |
| 6 | 加权分数 | 默认 0.5/0.3/0.2,可在 config.yaml 调 |
| 7 | 报告频率 | 每次复习自动出会话报告;周报留口子不实现 |
| 8 | 图谱形态 | 静态 HTML(主)+ 终端简版(辅);Streamlit 后期可选 |
| 9 | 阈值与配置 | 全部魔法数字提取到 config.yaml |
| 10 | 知识图谱 | NetworkX + PyVis,4 种关系类型,根因扩散诊断 |

---

## 十三、已交付的设计产物

- ✅ `srs-extraction-prompt.md` v1.1(完整可用,已含 7 种 pattern + 配额 + 自检清单)
- ✅ 本设计总览文档
- ⏳ 项目骨架代码(待实现,交给 Cursor)
- ⏳ config.yaml 模板(待实现)
- ⏳ 各模块实现(待实现)

---

## 十四、数据生命周期

本节定义所有"内容"和"状态"在不同操作下的更新规则。**实现 loader.py 时务必严格遵守。**

### 14.1 三类数据的更新模式

| 数据类别 | 存储 | 更新模式 | 谁产生 |
|---|---|---|---|
| **内容**(概念定义、题干、关系) | YAML 文件 | 你手编 / Claude 重抽取 | 你 + Claude |
| **状态**(作答历史、调度、统计) | SQLite | 系统自动写,只增不删 | 系统 |
| **配置**(权重、阈值、调度参数) | config.yaml | 你随时改 | 你 |

**核心原则:内容变化不应清零状态。**ID 不变 = 同一实体 = 历史保留。

### 14.2 同步规则(srs sync)

`srs sync` 是 YAML → SQLite 的对账操作,**默认 upsert + 不删除**。

**Upsert 行为**:

| 检测情况 | 动作 | 影响 |
|---|---|---|
| YAML 有,DB 无(新概念/新题) | INSERT | concept_stats 给该概念建空行 |
| YAML 有,DB 有,内容相同 | 跳过 | 无 |
| YAML 有,DB 有,内容不同 | UPDATE 内容字段 | **保留所有统计数据**(review_log、concept_stats、调度状态) |
| YAML 无,DB 有(孤儿) | 默认跳过,`--prune` 时标记 disabled | 历史保留,不再调度 |

**关键约束**:
- UPDATE 时**只改内容字段**(definition、key_facts、relations、题干、选项、解析等),**不动**统计字段(total_reviews、weighted_score、next_due_at 等)。
- `--prune` 也**不真删**,只把 `disabled = true`。review_log 永远保留。
- `--dry-run` 必须输出"会插入 X 条、更新 Y 条、标记孤儿 Z 条"的预演报告。

### 14.3 概念 ID 的稳定性约定

概念 ID 是跨时间的唯一身份证,必须稳定:

- **ID 不变 = 同一概念**,即使 definition 改了
- **ID 改了 = 新概念**,旧 ID 走孤儿处理路径

如果你想"重命名"概念(比如 `attention` → `attention-general` + 新建 `self-attention`),正确做法:
1. 在 YAML 里把老概念 ID 改名
2. 加新概念
3. `srs sync` 会把老 ID 当孤儿处理(disabled),新 ID 当新概念(空白统计)

**重大重定义检测**:`srs sync --strict-rename` 时,如果某概念的 definition 字段相比 DB 中的版本改动幅度过大(简单字符串相似度 < 70%),停下来询问:"⚠️ 概念 X 的定义有重大变化,这是修订(保留统计)还是重定义(应该用新 ID)?"。默认 sync 不做这个检测,因为大多数修改是合理修订。

### 14.4 题目 ID 的生成规则

**MVP 方案:题目 ID 用题干内容的 hash**(不再依赖 Claude 随机生成)。

```python
def question_id(question: dict) -> str:
    # 把题干 + 所有选项文本拼起来,去掉空白和标点
    raw = question["question"] + "".join(opt["text"] for opt in question.get("options", []))
    normalized = re.sub(r"\s+", "", raw)
    return "q-" + hashlib.sha256(normalized.encode()).hexdigest()[:8]
```

**好处**:
- 同样的题永远是同一个 ID,Claude 重抽取时实质相同的题自动去重
- 题干改了 ID 就变了,自然成为新题(老题作为孤儿被 disabled,不污染统计)

**注意**:抽取 prompt 现在让 Claude 自己生成 ID,但 loader 应该**忽略 Claude 给的 ID,强制按 hash 重算**——把 prompt 给的 ID 当占位符即可。这样不依赖 Claude 的一致性。

### 14.5 删除语义(永远不真删)

| 操作 | 物理动作 | 历史影响 |
|---|---|---|
| 标记题目质量 🚮 | `questions.disabled = true` | review_log 保留,调度跳过 |
| `srs sync --prune` 清孤儿 | 同上 | 同上 |
| 概念被弃用 | `concepts.disabled = true` | concept_stats 保留,图谱可选灰显 |

任何情况下都不 `DELETE FROM`,只 `UPDATE SET disabled = true`。

**唯一的例外**:`srs init --reset` 是显式删库重建,数据全清。这是 MVP 阶段的 schema migration 方案——以后真有用户数据要保护时,再引入 alembic。

### 14.6 统计重算策略

`concept_stats` 是**派生数据**,可以从 review_log 完整重建。

**MVP 全量重算**:
- 触发时机:每次 `srs review` 会话结束、`srs report` 生成前、`srs sync --rebuild-stats`、配置文件改动后下次访问
- 范围:扫一遍 review_log,聚合到 concept 维度
- 性能:10 万条 review_log 在 SQLite 里秒级完成,够用很久

**计入逻辑**:
- 已 disabled 的题的 review 记录**仍然计入** accuracy(诚实记录历史正确率)
- 但 disabled 题**不影响调度优先级**(不会再出现)
- 已 disabled 的概念在统计中标记但**不参与图谱可视化**(可选灰显)

### 14.7 配置变更不需要数据迁移

`config.yaml` 里的所有数值(权重、阈值、调度参数)都是**计算时使用**,不存进数据库。

- 改了 `weights.accuracy` 0.5 → 0.7:下一次 stats 计算时直接生效
- 改了 `thresholds.weak` 0.5 → 0.6:下一次掌握度判定时直接生效
- 改了 `scheduling.unknown_interval_hours` 6 → 4:下一道答"不知道"的题按新值

**永远不需要写迁移脚本来响应配置变更。**

### 14.8 关系图变更的传播

概念之间的 `relations` 改了(YAML 里加 / 删 / 改一条边),`srs sync` 时:
- 直接覆盖 DB 中该概念的 relations 字段
- 不影响任何统计
- 下次生成图谱或做根因扩散诊断时,自动按新关系跑

### 14.9 数据生命周期速查表

```
内容(YAML)─────────────────────────┐
   ↓ srs sync (upsert)              │ git 版本控制兜底
  DB 内容字段更新,统计保留          │ 出错可 git revert
                                    │
状态(DB)─────────────────────────────┘
   ↓ 永不删除,只 disabled
   ↓ 永远 append,从不 modify(review_log)
   
配置(config.yaml)
   ↓ 实时计算,不写库
   ↓ 改了立即生效

崩溃恢复:
   - YAML 损坏 → git 找回
   - DB 损坏 → 删库,knowledge/ 重新 sync,review 历史丢失
   - 因此建议:每月手动 export review_log 到 CSV 备份
```

---

## 十五、风险与已知约束

1. **LLM 出题质量参差**——必须配 👍/👎/🚮 反馈机制淘汰烂题(MVP 后加)
2. **概念 ID 跨笔记一致性**——多篇笔记共用概念时,Claude 不知道"这个概念之前定义过"。MVP 阶段:每篇笔记独立抽取,sync 时按 ID 自动合并(后到的 UPDATE 先到的)。后续可在抽取 prompt 里喂"已有概念清单",让 Claude 主动复用 ID。
3. **数据库迁移**——schema 改动时 MVP 用"删库重建"应付,review_log 提前导出 CSV;稳定后引入 alembic。
4. **跨设备同步**——`data/srs.db` 不进 git(避免冲突),设备间不能简单同步;后期引入 export/import JSON 命令解决。
5. **大笔记单次抽取超 LLM 上下文**——prompt 已支持分批输出,ID 一致性靠 hash 自动解决,不再需要人工把关。
6. **题目 hash 碰撞**——SHA-256 取 8 位,理论碰撞率约 1/40 亿,MVP 完全够用;真碰撞了 loader 报错,手动改题干即可。

---

**总览结束。**  
看完后告诉我:
1. 哪些地方理解有偏差需要修正?
2. 哪些设计决策想再讨论?
3. 还有没有漏掉的需求?

确认后即可把本文档 + `srs-extraction-prompt.md` v1.1 一并交给 Cursor 开始实现。
