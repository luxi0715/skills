# 工程项目方法论 · 通用版

> 从 legal-agent 项目沉淀,适用于任何长周期(50-200 小时)工程项目.
> 三层结构:通用纪律 / AI Agent 通用模式 / 具体场景特化.

---

## 目录

- [L1 · 9 条通用工程纪律](#l1--9-条通用工程纪律)
- [里程碑通用模板](#里程碑通用模板)
- [通用设计模式速查](#通用设计模式速查)
- [常见陷阱清单](#常见陷阱清单)
- [L2 · AI Agent 通用方法论](#l2--ai-agent-通用方法论)
- [L3 · CI Fix Agent 专属路线图](#l3--ci-fix-agent-专属路线图)
- [简历转化方法](#简历转化方法)

---

## L1 · 9 条通用工程纪律

### 1. 里程碑驱动,不要漂

每个里程碑必须满足 4 条:

- 工作量 ≤ 1 天(8 小时)
- 完成时能演示"用户感知到的能力"
- 写一份 evaluation.md
- 打一个 git tag(v0.x-xxx)

超过 1 天 → 拆.少于 2 小时 → 合.评测和 tag 是"完成"的必要条件,**不写就不算完**.

### 2. 决策点先行

每个里程碑开头花 5-10 分钟列 5 个决策.每个决策给:

- 3 个选项(A/B/C)
- 一句话理由
- 推荐选项

强制 5 分钟内拍板,拍完不回头.这一步省 5-10 小时"做到一半发现选错了".

### 3. 不超前(Just-In-Time)

要做的事:这一步 + 下一步.不做的事:再下一步.

里程碑 M5 在做时,M6 方案只列 1 行预告,不写代码,不深入讨论.超前规划 80% 会因前面发现新事实而改.

### 4. 不重写

出现"要不要重写"念头时,先回答 3 个问题:

- 现状的具体问题是什么(列 3 条)
- 重写能解决哪几条
- 重写引入哪些新风险

如果重写解决 ≤ 2 条 + 引入新风险 ≥ 2 条 → 不重写,改进现状.

### 5. 单例 + 急加载(连接池标准模式)

任何数据库 / 第三方服务连接,统一这套:

```python
_client = None

async def init_xxx() -> XxxClient:
    global _client
    if _client is None:
        _client = create_xxx_client(...)
        await _client.health_check()  # 急加载,启动时立即握手
    return _client

async def close_xxx() -> None:
    global _client
    if _client:
        await _client.close()
        _client = None

def get_xxx() -> XxxClient:
    if _client is None:
        raise RuntimeError("xxx not initialized")
    return _client
```

3 个函数,全项目所有外部依赖统一这套.急加载是关键:不做的话第一次调用慢 10-30 秒.

### 6. 三层架构(API / Service / Component / Data)

```
Entry 层    HTTP / CLI / cron (对外接口)
   ↓
Service 层  业务编排(把组件串起来)
   ↓
Component   单一职责的工具(检索 / 抽取 / 解析)
   ↓
Data 层     PG / Redis / 缓存 / 队列
```

新人项目最容易死在没分层 —— 业务逻辑写在端点里,3 个月后改不动.

### 7. 评测体系优先于功能数量

宁可 5 个能力 + 完整评测,不要 10 个能力 + 零评测.

每个里程碑收尾必须能回答:

- 这次改动相比上次,**哪个指标**变好
- 数字是多少(具体到小数点)
- 怎么测的

没有数字 → 这个里程碑没完成.

### 8. 提交粒度 = 里程碑

git commit / push / tag 都按里程碑.单个里程碑内的多次小提交在收尾时 squash 成 1 个.commit message 详细写改了什么、为什么改、数字效果.

简历评审看的就是 commit message + tag 演化,这是工程能力最直接证据.

### 9. 文档先写"为什么",再写"怎么做"

每个 evaluation.md 必须有 3 段:

- **实验设置**:工具版本、参数、数据规模
- **数据结论**:具体数字 + 表格
- **工程发现**:踩坑、设计取舍、未来优化空间

"为什么这么选"比"代码长什么样"重要 10 倍 —— 代码 git 里有,选型理由不写就忘.

---

## 里程碑通用模板

每个里程碑严格按 3 步走.

### Step 1:决策点(5-10 分钟)

```markdown
# M.x.0 决策

## 决策 1: 数据存储
A. 选项 A → 一句话理由
B. 选项 B → 一句话理由
C. 选项 C → 一句话理由
推荐:B

## 决策 2-5
...

## 拍板
最终:BACMγ
```

### Step 2:实施(主要时间)

- 拆 3-5 个子步骤(M.x.1 / M.x.2 / ...)
- 每个子步骤:cat heredoc 写代码 + 跑测试验证
- 出现问题修一次,不重写
- 工具栈固定:一个里程碑内不切语言、不换框架

### Step 3:收尾(30-60 分钟)

```bash
# 1. 写评测文档
cat > docs/notes/Mx-evaluation.md << 'EOF'
... 实验设置 / 数据结论 / 工程发现 ...
EOF

# 2. 三连
uv run ruff format <paths>
uv run ruff check <paths> --fix
uv run mypy <paths>

# 3. git
git add -A
git status
git commit -m "feat(Mx): xxx"
git tag v0.x-name
git push origin main --tags
```

---

## 通用设计模式速查

3 个模式贯穿大多数项目.

### 模式 A:Strategy Pattern(策略模式)

适用:同一能力,多种实现可切换.

```python
class Xxx(ABC):
    @abstractmethod
    async def execute(...) -> Result: ...

class Impl1(Xxx): ...
class Impl2(Xxx): ...

def get_default_xxx() -> Xxx:
    return Impl2()
```

例子:Router(规则/LLM/级联)、Retriever(向量/BM25/混合)、Generator(模板/LLM).

### 模式 B:Cascade Pattern(级联模式)

适用:"便宜"和"准确"是天敌时.

核心:置信度高直接出,置信度低升级到更贵的方法.

例子:

- Cascade Router(70% 规则 0ms + 30% LLM 2s)
- 缓存级联(本地 → Redis → DB)
- 模型级联(小模型先试 → 复杂任务升级大模型)

### 模式 C:Fire-and-Forget(异步剥离)

适用:主流程有"用户不关心结果"的副作用任务.

```python
asyncio.create_task(some_background_task())
# 不 await,主流程立即继续
# 错误隔离用 logger.exception 包
```

例子:异步抽 entity(record_turn 50x 提速)、日志写文件、监控上报、缓存预热.

注意:不能用于"用户在等结果"的操作.

---

## 常见陷阱清单

每条都是真实踩过的坑.

### 陷阱 1:Windows + 异步 PG 客户端

```
psycopg async 不兼容 Windows ProactorEventLoop
症状:Application startup failed - InterfaceError
解法:
  if sys.platform == "win32":
      asyncio.set_event_loop_policy(WindowsSelectorEventLoopPolicy())
注意:必须在 import 任何异步库之前
生产用 Linux 无此问题
```

### 陷阱 2:redis-py 5.x 类型签名

```
所有方法返回 Awaitable[T] | T(同异步共用签名)
mypy 报 "Incompatible types in await"
解法:await client.xxx()  # type: ignore[misc]
```

### 陷阱 3:Cypher vs SQL 字符串函数

```
SQL:    substr(s, start, length)    start 1-indexed
Cypher: substring(s, start, len)    start 0-indexed
切换查询语言时函数名 + 索引规则都不一样
```

### 陷阱 4:首次连接握手延迟

```
症状:第一次调用慢 10-30 秒,之后正常
原因:连接池懒加载,第一次真正建立 TCP + 认证
解法:init_xxx 里强制做一次 ping / verify_connectivity
影响:legal-agent 的 record_turn 23s → 965ms 全靠这个
```

### 陷阱 5:Lost in the Middle

LLM 对 prompt 头尾敏感,中间内容容易被忽略.重要内容放头或尾,次要的放中间.RAG 场景特别明显.

### 陷阱 6:LLM 工具描述写不清,LLM 选错工具

工具 description 必须:
- 第一句话讲核心用途(LLM 看头部决策)
- 列举正面例子("适用于:...")
- 列举反面例子("不适用:请用 XX 工具")
- 参数名 + 描述要明确格式(中文数字 / 阿拉伯)

### 陷阱 7:粘贴命令到 bash 时回车被吞

heredoc 多行命令最好放在脚本文件里跑,不要直接粘 terminal.如果必须粘,用单个完整命令,避免分多行.

### 陷阱 8:Docker 容器启动慢于服务初始化

服务启动时容器可能还在 starting 状态.解决:容器 healthcheck + 应用层 init_xxx 带重试.

---

## L2 · AI Agent 通用方法论

AI Agent 项目特化模式.做任何 LLM Agent 都用得到.

### 1. LM-Centric ACI 设计(工具空间)

给 Agent 套裸 bash 是新手错误.正确做法是设计**受约束工具空间**.

**核心原则**:

```
原则 1: 输出格式固定
  view_file 永远返回带行号的 window(默认 100 行)
  不让 LLM 应对 ls 几百个文件那种灾难输出

原则 2: 失败要给替代建议
  search 没匹配 → 返回"未匹配,最相近的 N 项是 X / Y / Z"
  不让 LLM 在不确定状态盲目继续

原则 3: 写入前预检
  edit 写入前过 linter / AST 检查,不通过直接拒绝
  避免脏代码污染下游

原则 4: 每个命令返回状态码 + 下一步建议
  让 LLM 知道"成功 / 部分成功 / 失败"
```

**工具命名规约**:

```
查看类:view_file / list_dir / search_dir
修改类:edit_file / create_file / delete_file
执行类:run_test / run_lint / run_shell(谨慎)
信息类:get_xxx_info / find_related_xxx
```

**参数设计原则**:

```
1. 必填参数尽量少(LLM 容易漏)
2. 可选参数有合理默认值
3. 参数类型用 enum 而不是 free text
4. 路径类参数统一相对路径(绝对路径暴露环境)
```

### 2. 多模态输入融合

输入可能是文本 + 截图 + 日志混合(CI Fix Agent 场景).

**融合策略**:

```
Step 1: 单模态解析
  • 文本:tree-sitter / 正则抽实体
  • 截图:Vision LLM 抽 OCR + UI 元素 + 高亮区域
  • 日志:stack trace 抽函数名 / 文件名 / 行号

Step 2: 实体对齐
  • 截图里的报错文案 ↔ 日志里 Exception message
  • 截图里的元素 ID ↔ 代码里的 DOM 选择器
  • 跨源相互印证 → 置信度加权

Step 3: 统一结构注入下游
  形成 {entities, evidence_sources, confidence} 三元组
  下游 Agent 看的是结构化数据,不是原始 mixed bag
```

### 3. 评测体系(SWE-bench-like)

没评测的 Agent 项目最大风险:"改了 prompt 不知道好坏,感觉变快了但回归率涨了".

**评测集设计 4 条原则**:

```
1. 任务来自真实场景
   不要人工编造,从历史数据回放
   • CI Fix:历史 CI 失败回放
   • 客服 Agent:历史工单
   • 法律咨询:真实用户提问日志

2. 每个任务含 F2P + P2P
   • F2P(Fail-to-Pass):修了 bug 应该通过的测试
   • P2P(Pass-to-Pass):之前能过的功能不能 break
   防止 Agent 为了 F2P 把 P2P 改坏

3. 测试 patch 对 Agent 隐藏
   不让 Agent 看到测试代码,防止"刷题式"作弊

4. 评测集规模 100-200 条够用
   规模太小代表性差,太大成本爆炸
   重点是覆盖长尾场景
```

**Trajectory 三阶段归因(TRAJEVAL 思路)**:

把 pass@1 单一数字拆解成:

```
search 阶段 P / R    找对文件了吗
read 阶段 P / R      读对了但理解错代码?
edit 阶段 P / R      改对了但破坏 P2P?
```

每次 prompt 改动后跑全量,生成失败分类报告.单一数字升降看不出问题在哪.

### 4. Self-correction 闭环

Agent 出错后,关键是**给它有用的反馈**,而不是简单"重试".

**好的反馈结构**:

```
错误反馈应包含 3 段:

1. 失败现象
   "patch 应用失败:在 file.py 第 42 行找不到匹配"

2. 最相近的可能
   "找到相似行,差异在:
    - 第 2 行的缩进(SEARCH 用 4 空格,实际是 tab)
    - 第 5 行变量名(SEARCH 是 user_id,实际是 userId)"

3. 下一步建议
   "建议:重读 file.py 第 38-50 行,确认实际格式后重新生成 patch"

→ 让 LLM self-correct,不是盲目重试
```

**重试预算**:

```
单任务最多 3 轮 self-correction
3 轮失败 → 标记需人工介入
不要无限重试 —— 4 轮以上失败率指数升高,只浪费 token
```

### 5. 沙箱与隔离

LLM 生成的代码必须强隔离运行.

**沙箱设计 5 条**:

```
1. 容器化(Docker)
   防止 rm -rf 误伤主机

2. 资源限制(cgroup)
   • 内存上限
   • PID 上限(防 fork bomb)
   • CPU 配额

3. 事件驱动架构(OpenHands V1 风格)
   每个 action(exec / edit / read)作为 immutable event
   写入 event stream
   沙箱崩溃可重启重放至最近状态

4. 分层镜像缓存
   • base layer:语言运行时 + 通用依赖(月级别更新)
   • per-repo layer:项目依赖(lockfile hash 缓存)
   • 重复任务跳过依赖装载

5. 推理路由(成本优化)
   • 简单任务走内部蒸馏 / 小模型
   • 复杂任务升级云端大模型
   • 单任务平均成本能降一个数量级
```

### 6. 模型/参数调参

**何时调参**:

- 决策点拍板完后不动框架,先用默认参数跑通
- 跑通后用评测集做小规模 ablation(2-3 个参数组合)
- 调参结果写入 evaluation.md,有数字支撑

**常调参数**:

```
LLM 侧:
  • temperature(决策类用 0,生成类 0.3-0.7)
  • max_tokens(限制输出长度,降本)
  • top_p / top_k(很少调)

Agent 侧:
  • 最大迭代轮数(防死循环)
  • 工具调用预算(防 token 爆炸)
  • 上下文 window 大小(view_file 默认 100 行)
  • history 保留策略(全部 / 截断 / 摘要)
```

### 7. 数据飞轮

失败 trajectory 不能扔,要进数据飞轮.

```
Step 1: tail-based 采样
  • 100% 失败 trajectory 保留
  • 5% 成功 trajectory 采样(控成本)

Step 2: TRAJEVAL 归因到阶段
  search / read / edit 哪个阶段挂的

Step 3: 沉淀为新评测集
  • 失败 case → 下次评测集的"难题"
  • 同时反推 ACI / prompt 优化方向
```

---

## L3 · CI Fix Agent 专属路线图

针对你接下来要做的项目,我把 L2 方法论映射到具体里程碑.

### 项目定位

```
输入:CI 失败的代码仓库 + 测试日志 + (可选)截图
输出:能让 F2P 测试通过 + P2P 不破的 patch
约束:全程在企业内网沙箱,代码不外传
对标:SWE-agent / Aider / OpenHands
```

### 里程碑路线图(13 个,~120 小时)

```
M0  脚手架
    uv + src 布局 + pytest + ruff + mypy
    docs/notes/ROADMAP.md 写完路线图
    git tag v0.0-scaffold

M1  最小 LLM 调用
    OpenAI SDK / 内部 LLM 网关
    单测验证能调通 + 流式
    git tag v0.1-llm

M2  数据底座
    PG(存 trajectory) + Redis(任务队列)
    sessions / tasks / trajectories 三表
    git tag v0.2-db

M3  仓库索引(单语言 MVP)
    tree-sitter 抽 Python definition + reference
    构建文件间引用图
    grep_ast 渲染省略代码视图
    Top-5 文件命中率评测
    git tag v0.3-repo-index

M4  Personalized PageRank 排序
    在 M3 引用图上跑 PPR
    chat 已加载文件 50x 权重 / issue 提及 10x / 私有符号扣权
    token budget 内二分搜索塞最重要 symbol
    git tag v0.4-pagerank

M5  ACI 工具空间
    view_file(path, line_range)
    search_dir(pattern, scope)
    edit_file(path, search, replace)
    run_test(test_id)
    输出格式固定 + 失败给替代建议
    对比 raw-bash baseline 评测
    git tag v0.5-aci

M6  SEARCH/REPLACE Diff 引擎 V1
    LLM 输出 SEARCH/REPLACE 块
    第一级 perfect_replace 精确匹配
    成功率 baseline
    git tag v0.6-diff-v1

M7  四级模糊匹配 Diff V2
    L1 精确 / L2 缩进容忍 / L3 省略号 / L4 difflib
    AST 预检
    失败时给详细错误反馈
    成功率从 ~60% 提升到 ~90%
    git tag v0.7-diff-v2

M8  Docker 沙箱
    base + per-repo 双层镜像
    cgroup 资源限制
    event stream 架构
    git tag v0.8-sandbox

M9  Self-correction 闭环
    测试反馈截短(head 20 + tail 20)
    最多 3 轮重试 + 转人工
    3 轮内收敛率评测
    git tag v0.9-self-correct

M10 多模态融合(可选,简历加分项)
    Vision LLM 抽截图
    实体对齐 stack trace
    git tag v0.10-multimodal

M11 评测集 + TRAJEVAL
    150 条历史 CI 失败回放
    F2P + P2P 双轨
    search / read / edit 三阶段归因
    git tag v0.11-eval

M12 LangGraph Agent 编排
    把 M5-M9 各组件接到 StateGraph
    thinker → search → read → edit → test → 循环
    git tag v0.12-langgraph

M13 LangSmith 监控 + 简易 UI
    所有 trajectory 上 LangSmith
    简单 web UI 看 trajectory 时间线
    git tag v0.13-observability
```

### 跟 legal-agent 的关键差异

```
1. 主输入是仓库不是 query
   仓库索引(M3 / M4)是 legal-agent 没有的核心能力

2. 工具空间更复杂
   legal-agent 3 个查询工具
   CI Fix 至少 5 个工具(view/search/edit/run_test/git)

3. 评测体系是核心生命线
   legal-agent 评测是"加分项"
   CI Fix 评测是"必需品"(没评测就是赌博)

4. 沙箱隔离是新东西
   legal-agent 不涉及代码执行
   CI Fix 必须强隔离

5. Diff 算法是新核心
   legal-agent 没有
   CI Fix 决定成功率的最大变量
```

### 时间预估

```
M0-M2  基建        ~6 小时
M3-M4  仓库索引     ~12 小时
M5     ACI         ~8 小时
M6-M7  Diff 引擎    ~16 小时
M8     沙箱        ~10 小时
M9     Self-correct ~6 小时
M10    多模态      ~10 小时(可选)
M11    评测体系    ~12 小时
M12    Agent 编排  ~8 小时
M13    监控 UI     ~10 小时

合计:90-110 小时
```

### 你 100 小时怎么分

```
方案 A:全做 MVP(每个 M 偷功减料)
  最终能跑通 + 评测数据 + 简历能讲
  风险:深度不够,面试细节问就穿

方案 B:挑深做(我推荐)
  必做:M0-M9 + M11 评测
  跳过:M10 多模态 + M13 监控 UI
  深做:M3 仓库索引 + M7 Diff V2 + M11 评测
  3 个深做的就是简历最强金句

方案 C:学习路径
  先做 M5 ACI 单点突破
  做到能跑就接到 LangGraph(M12)
  最后补 M3 索引 + M11 评测
```

---

## 简历转化方法

每个里程碑做完,立即按这套模板写"简历金句素材",放在 evaluation.md 末尾.

### 金句模板

```
[能力名] · [量化数字]

技术栈:
  • 核心库 A、B、C(列 3-5 个)
  • 数据库 / 框架(列 2-3 个)

做了什么(150-300 字):
  问题陈述 + 设计取舍 + 实施细节 + 量化结果
  
关键决策:
  • 选了 X 没选 Y 的理由
  • 跟主流方案的差异点

简历金句(50-80 字一句):
  "用 XX 实现 YY,在 ZZ 数据集上 P 指标从 X% 提升到 Y%"
```

### 好金句的 5 个特征

1. **有数字**:"召回率从 47% 到 78%",不是"显著提升"
2. **有对照**:"vs raw-bash baseline +18.3 pp"
3. **有具体技术**:"tree-sitter + personalized PageRank",不是"用了一些算法"
4. **有取舍**:"选 SEARCH/REPLACE 而非 unified diff,因为...",不是堆砌
5. **有规模感**:"60+ 语言 / P99 < 80ms / 5000+ 内部用户"

### 反例(空话金句)

```
❌ "使用 LangChain 框架构建 LLM Agent"
❌ "实现了高性能的代码检索系统"
❌ "采用先进的多模态融合技术"

→ 没数字 / 没技术 / 没场景 → 没人信
```

### 好例(legal-agent 的金句)

```
✅ "Cascade Router:Rule 优先 + LLM 兜底,
   70% query 规则 0ms 命中,30% LLM 兜底,
   平均延迟从 2685ms 降至 750ms,
   准确率保持 100%"

✅ "M9.1 异步 Entity 抽取:
   record_turn 主流程从 23 秒降至 965ms,
   ~50x 提速.关键发现:阻塞源是 Redis 客户端
   首次握手而非 entity 抽取本身,
   通过 init_redis 强制 ping() 完成握手前置启动成本"
```

### 项目级简介模板

简历"项目经历"段落写法:

```
[项目名]· [一句话定位]
技术栈: [核心 5-8 个]

主要内容: [项目目标 + 你的角色 + 量化规模]

[亮点 1 标题]: [量化数字]
  [3-5 行展开:问题 + 设计 + 结果]

[亮点 2 标题]: [量化数字]
  [3-5 行展开]

[亮点 3 标题]: [量化数字]
  [3-5 行展开]

[局限与下一步]:
  [展示对项目边界的认知]
```

你 CI Fix Agent 描述就是这个模板的完美范例,严格按这个写.

---

## 附录:legal-agent 沉淀的具体模式

如果你做的是检索 / 对话类项目,legal-agent 这些模式可以直接复用.

### 三层记忆架构(对话类项目)

```
Buffer Memory(Redis 滑窗,最近 N 轮)
Summary Memory(PG,历史压缩)
Hard Memory(PG,结构化用户档案)

inject_memory_into_messages 编排注入
record_turn 编排更新
```

### Hybrid Search(检索类项目)

```
向量检索(语义相似)+ BM25(关键词)
RRF(Reciprocal Rank Fusion)融合
Two-stage:recall 50 → rerank 5
```

### Persona Guard(对话产品)

```
yaml 配置 N 套 persona
全局禁词 + 单 persona 禁词
事后审计 + log warning + 用户级追责
完整版升级:LLM-as-Judge 替换关键词
```

不在你 CI Fix Agent 范围内的就跳过.

---

## 用法清单

新项目第一天:

1. clone 你的 项目脚手架模板
2. 把这个 methodology.md 复制到 docs/methodology.md
3. 写 docs/notes/ROADMAP.md 列里程碑路线图
4. 把 ROADMAP 给 AI 助手(配合 project-skill-prompt.md)
5. 进 M0 决策点 → 实施 → 收尾 → tag

每个里程碑开始时:

1. 写 docs/notes/Mx-decisions.md
2. 列 5 个决策,5 分钟拍板
3. cat heredoc 实施
4. 跑测试验证
5. 收尾三连 + git tag

每两周:

1. 看 git log --oneline 自己的里程碑节奏
2. 慢了就拆任务,快了就检查是否偷工
3. 更新简历金句素材文档
