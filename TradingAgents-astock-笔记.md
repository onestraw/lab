项目地址：https://github.com/simonlin1212/TradingAgents-astock

跑起来：streamlit run web/app.py

Streamlit 是一个开源的 Python 库，专为数据科学家、机器学习工程师和开发者设计。它允许开发者仅使用纯 Python 脚本，在无需编写任何前端代码（如 HTML、CSS 或 JavaScript）的情况下，即可快速构建出精美的交互式 Web 应用程序。

# 介绍

我来从整体到局部系统地介绍这个项目的架构和设计思路。现在让我获取关键的架构文件来深入理解设计：现在我为你详细介绍这个项目的架构和设计思路。

## 整体架构设计

### **核心理念：多Agent投研框架的A股深度特化**

TradingAgents-Astock 是基于原版 TradingAgents 框架的深度改造，核心思路是**从数据、角色、规则三个维度** 完全适配A股特征：

```
用户输入(股票代码+日期)
        ↓
  [7个分析师并行分析] ← 数据层(零第三方库)
        ↓
  [Bull vs Bear投研辩论] ← A股特化逻辑
        ↓
  [三方风险辩论] ← 激进/保守/中立
        ↓
  [Portfolio Manager综合决策] ← 深度思考LLM
        ↓
  Buy/Hold/Sell + 仓位建议
```

---

## 架构分层详解

### **1. 数据层（Data Layer）— 零第三方数据库依赖**

这是本项目最核心的创新。所有数据通过直连HTTP API获取，完全避免了第三方积分墙（如Tushare）：

| 数据源 | 协议 | 优势 | 用途 |
|--------|------|------|------|
| **mootdx** | TCP 7709 | 通达信源、不封IP | K线、财务快照、F10文本 |
| **腾讯财经** | HTTP (qt.gtimg.cn) | 实时、GBK编码 | PE/PB/市值/换手率 |
| **东财push2** | HTTP | 独家龙虎榜、解禁数据 | 龙虎榜、限售、板块行情 |
| **新浪财经** | HTTP | K线备用源 | 财报三表(资产负债表/利润表/现金流) |
| **同花顺** | HTTP (10jqka) | EPS一致预期、热股 | 分析师预期、概念题材 |
| **财联社** | HTTP (cls.cn) | 全球财经快讯 | 宏观政策动向 |

**关键创新：东财防封机制（v0.2.11）**

```python
def _em_get(url, params=None, ...):
    """东财统一请求入口"""
    wait = _EM_MIN_INTERVAL - (time.time() - _em_last_call[0])
    if wait > 0:
        time.sleep(wait + random.uniform(0.1, 0.5))  # 串行限流 + 随机抖动
    try:
        return _EM_SESSION.get(url, ...)  # 复用session(Keep-Alive)
    finally:
        _em_last_call[0] = time.time()
```

东财接口在多Agent高频请求时容易被风控临时封IP，所以所有7个调用点（push2/push2his/datacenter-web/search-api/np-weblist）都走`_em_get()`，实现：
- **串行限流**：`EM_MIN_INTERVAL=1.0s`（可设环境变量调整）
- **随机抖动**：0.1~0.5s，规避机器人检测
- **Keep-Alive复用**：减少连接开销

---

### **2. Agent层（Agent Layer）— 7个分析师角色**

#### **原版4个Analyst（已A股化）**

1. **🏪 市场分析师 (Market Analyst)**
   - 分析：K线形态、技术指标、量价关系
   - 工具：`get_stock_data`, `get_indicators`
   - 输出：支撑阻力、趋势评价

2. **💬 舆情分析师 (Social Media Analyst)**
   - 分析：散户情绪、论坛热度
   - 工具：`get_news`
   - 输出：市场情绪指数

3. **📰 新闻分析师 (News Analyst)**
   - 分析：行业新闻、公告、宏观事件
   - 工具：`get_news`, `get_global_news`, `get_insider_transactions`
   - 输出：事件风险评估

4. **📊 基本面分析师 (Fundamentals Analyst)**
   - 分析：财报、盈利能力、估值
   - 工具：`get_fundamentals`, `get_balance_sheet`, `get_cashflow`, `get_income_statement`
   - 输出：内在价值、增长潜力

#### **A股特化3个Analyst（新增）**

5. **🏛️ 政策分析师 (Policy Analyst)**
   - **为什么A股需要**：A股是"政策市"，政策变化直接驱动板块轮动
   - 分析：监管政策、产业政策、窗口指导
   - 工具：`get_news`, `get_global_news`
   - 输出：政策导向、风险预警

6. **🔥 游资追踪师 (Hot Money Tracker)**
   - **为什么A股需要**：游资是短线定价的核心力量
   - 分析：龙虎榜动向、大单流向、主力资金动态
   - 工具：`get_dragon_tiger_board`, `get_fund_flow`, `get_concept_blocks`, `get_northbound_flow`
   - 输出：资金流向、短线热点

7. **🔓 解禁监控师 (Lockup Watcher)**
   - **为什么A股需要**：解禁是A股特有的重大供给冲击
   - 分析：限售股解禁、大股东减持、股权质押
   - 工具：`get_lockup_expiry`, `get_insider_transactions`
   - 输出：解禁压力、风险等级

---

### **3. 决策层（Decision Layer）— 双LLM架构**

项目采用**双LLM设计**，分离快速和深度思考：

```python
# quick_think_llm 用于所有Analyst
market_analyst = MarketAnalyst(llm=quick_thinking_llm)
social_analyst = SocialMediaAnalyst(llm=quick_thinking_llm)
# ... 所有7个Analyst都用快速LLM

# deep_think_llm 用于需要综合全局的决策
research_manager = ResearchManager(llm=deep_thinking_llm)
portfolio_manager = PortfolioManager(llm=deep_thinking_llm)
```

**为什么这样设计？**
- 快速LLM：处理单个分析任务（K线识别、新闻总结），响应快、成本低
- 深度LLM：综合7份报告、多轮辩论结果，做最终投资决策，需要复杂推理

---

### **4. 辩论层（Debate Layer）— 多轮制衡**

#### **第一轮：投研辩论 (Investment Debate)**
```
7个分析师报告
        ↓
Bull Researcher ←→ Bear Researcher (最多N轮)
        ↓
Research Manager综合判决（深度思考）
```

Bull方主张买入，Bear方主张卖出，通过对抗性论证逼出更全面的分析。

#### **第二轮：交易方案 (Trader)**
```
Research Manager的投资计划
        ↓
Trader应用A股约束条件：
  - T+1制度（今天买明天才能卖）
  - 涨跌停限制（±10%为上限）
  - 最小手数（100股=1手）
  - 交易时段（9:30-11:30, 13:00-15:00）
        ↓
生成可执行交易方案
```

#### **第三轮：风险三方辩论 (Risk Assessment)**
```
Aggressive Risk Debater (主张激进加仓)
        ↓
Conservative Risk Debater ←→ Neutral Risk Debater (主张保守、中立)
        ↓
Portfolio Manager最终决策（深度思考）
```

三方从不同风险偏好出发，Portfolio Manager综合判断最终的Buy/Hold/Sell信号和仓位。

---

### **5. 执行流程（Workflow）— LangGraph编排**

```python
class TradingAgentsGraph:
    def propagate(self, company_name, trade_date):
        return self._run_graph(company_name, trade_date)
```

内部通过 LangGraph 编排状态机：

```
初始化 → Analyst并行 → 质量门控 → Bull/Bear辩论 → Research Manager
    ↓
Trader交易方案 → Risk三方辩论 → Portfolio Manager → 最终决策 → 报告导出
```

**关键设计：状态机模式**
- 每个节点是一个Agent或Manager
- 状态对象`AgentState`贯穿全过程，包含：
  - 7份分析师报告
  - 投研辩论历史
  - 交易方案
  - 风险评估
  - 最终决策

---

## 核心模块详解

### **模块1：数据流（`a_stock.py`，2100行）**

关键函数映射到17个工具方法：

```python
# 基础行情
get_stock_data()          # OHLCV K线
get_indicators()          # 技术指标(RSI/MACD/Bollinger)

# 基本面
get_fundamentals()        # PE/PB/市值
get_balance_sheet()       # 资产负债表
get_cashflow()            # 现金流量表
get_income_statement()    # 利润表
get_profit_forecast()     # EPS一致预期

# 新闻舆情
get_news()                # 个股新闻
get_global_news()         # 全球财经快讯

# 游资追踪（A股特化）
get_dragon_tiger_board()  # 龙虎榜
get_fund_flow()           # 资金流向（分钟+日级）
get_northbound_flow()     # 沪深股通
get_hot_stocks()          # 热股题材

# 解禁监控（A股特化）
get_lockup_expiry()       # 限售解禁日历
get_insider_transactions()# 股东变化

# 其他
get_concept_blocks()      # 概念板块
get_industry_comparison() # 行业对比
```

**中文股票名自动解析链路**：

```python
用户输入："宝光股份"
    ↓
safe_ticker_component() 检测中文字符
    ↓
resolve_ticker() 调用_build_name_code_map()
    ↓
mootdx 全市场映射(缓存) → {"宝光股份": "600379"}
    ↓
返回 6位代码 "600379"
```

### **模块2：图设置（`setup.py`）**

定义LangGraph的拓扑和节点：

```python
GraphSetup.setup_graph(selected_analysts):
    添加Analyst节点
    添加Researcher节点(Bull/Bear)
    添加Manager节点(Research + Portfolio)
    添加Risk Debaters节点(Aggressive/Conservative/Neutral)
    定义节点间的边(routing logic)
```

### **模块3：状态传播（`propagation.py`）**

```python
class Propagator:
    def create_initial_state(company_name, trade_date, past_context):
        # 初始化AgentState，包含交易记忆日志
        # 这样分析师能看到"上次对这只股票的预测如何表现"
```

### **模块4：反思机制（`reflection.py`）**

```python
class Reflector:
    def reflect_on_final_decision(final_decision, raw_return, alpha_return):
        # 对比预测vs实际，生成反思报告
        # Alpha基准：CSI 300（沪深300）
```

---

## 设计模式精妙处

### **1. 安全边界（`safe_ticker_component`）**

```python
def safe_ticker_component(ticker: str) -> str:
    """路径安全校验 + 中文自动转码"""
    # 阻止 "../../../../etc/passwd" 这类攻击
    # 自动把中文股票名转成6位代码
```

这是一个关键的防御机制，确保ticker无论来自用户输入还是LLM输出，都被正规化和清理。

### **2. 数据源优先级**

```
高优先级（不封IP）：mootdx(TCP) / 腾讯 / 新浪 / 同花顺
    ↓ fallback
中优先级（限流）：东财(push2/datacenter)
    ↓ 最后才用
低优先级：百度(已下线)
```

这种分层策略让系统既能获取所有必需数据，又不会频繁触发东财风控。

### **3. 检查点与断点续跑**

```python
checkpoint_enabled: 支持SQLite保存中间状态
    → 如果12阶段pipeline在第8步崩溃
    → 下次运行自动从第8步恢复（不重复前7步）
```

这对长链路很关键，因为一次完整分析涉及30-50次LLM调用，网络波动时很容易失败。

### **4. 交易记忆日志**

```python
memory_log.store_decision(ticker, date, signal)
    ↓ N天后
memory_log.get_pending_entries()  # 获取待反思的决策
    ↓
reflector.reflect_on_final_decision()  # 对比实际收益
```

系统能从自己的历史决策中学习，这是真正的"Agent回顾"。

---

## 数据流示意

```python
# 用户调用
ta = TradingAgentsGraph(config={...})
final_state, decision = ta.propagate("688017", "2026-05-12")

# 内部流程
1. Analyzer初始化
   - 加载LLM (deep_think + quick_think)
   - 创建工具节点(ToolNode)
   - 编译LangGraph

2. 状态初始化
   - 读取过去交易记忆
   - 创建AgentState对象

3. 7个Analyst并行执行
   - Market Analyst:   get_stock_data + get_indicators
   - Social Analyst:   get_news (舆情)
   - News Analyst:     get_news + get_global_news + get_insider_transactions
   - Fundamentals:     get_fundamentals + get_balance_sheet + ...
   - Policy:           get_news + get_global_news
   - Hot Money:        get_dragon_tiger_board + get_fund_flow + ...
   - Lockup:           get_lockup_expiry + get_insider_transactions

4. Bull vs Bear辩论
   - Bull Researcher读7份报告，主张买入
   - Bear Researcher反驳，主张卖出
   - 最多N轮对话

5. Research Manager综合判决
   - 深度思考LLM
   - 输出投资计划(理由+风险)

6. Trader应用约束
   - 检查T+1/涨跌停/手数
   - 生成可执行交易方案

7. Risk三方辩论
   - Aggressive: 加仓! (收益 +20%, 风险 +30%)
   - Conservative: 不动! (稳定, 但错过机会)
   - Neutral: 小仓位 (平衡)

8. Portfolio Manager最终决策
   - 深度思考LLM
   - 输出Buy/Hold/Sell + 仓位(0%/50%/100%)

9. 决策记忆存储
   - memory_log.store_decision()
   - 下次同ticker运行时会对比收益

10. 报告导出
    - Markdown (零依赖)
    - PDF (中文字体自适配)
```

---

## 配置灵活性

```python
config = {
    "llm_provider": "minimax",           # 9个供应商可选
    "deep_think_llm": "MiniMax-M2.7",    # 深度思考模型
    "quick_think_llm": "MiniMax-M2.7-highspeed",  # 快速模型
    "output_language": "Chinese",        # 报告语言
    "max_debate_rounds": 2,              # 辩论轮数
    "max_risk_discuss_rounds": 1,        # 风险讨论轮数
    "checkpoint_enabled": True,          # 断点续跑
    "data_cache_dir": "~/.tradingagents/cache",
    "results_dir": "~/.tradingagents/results",
}
```

---

## 总结：架构的三大创新点

1. **零依赖数据层**：绕过积分墙，全部直连HTTP API，代码即使有问题也能独立修复
2. **A股特化Agent**：政策/游资/解禁三个角色是美股框架完全没有的
3. **多层制衡决策**：投研辩论→交易约束→风险三方，三层制约确保决策鲁棒性

这个架构把一个通用的多Agent框架，通过系统化的改造，真正落地到了A股的规则和现实。
