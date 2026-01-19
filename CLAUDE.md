# CLAUDE.md

此文件为 Claude Code (claude.ai/code) 在本仓库中工作时提供指导。

## 项目概述

A股智能分析系统 - 基于 AI 大模型的 A/H 股自选股智能分析系统，每日自动分析并推送「决策仪表盘」到企业微信/飞书/Telegram/邮箱。使用 Python 3.10+ 构建，采用 Google Gemini AI 进行分析，支持 GitHub Actions 零成本部署。

## 核心命令

### 开发与测试
```bash
# 运行单次分析（不发送通知）
python main.py --no-notify

# 调试模式（详细日志）
python main.py --debug

# 仅获取数据，不进行 AI 分析
python main.py --dry-run

# 分析指定股票（覆盖配置文件）
python main.py --stocks 600519,300750,002594

# 单股推送模式（每分析完一只立即推送）
python main.py --single-notify

# 仅运行大盘复盘
python main.py --market-review

# 跳过大盘复盘
python main.py --no-market-review

# 启动定时任务模式
python main.py --schedule

# 启动 WebUI 管理界面
python main.py --webui
```

### 环境配置
```bash
# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate   # Windows

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env 填入 API Key 等配置
```

### Docker 部署
```bash
# 构建并启动
docker-compose up -d

# 查看日志
docker-compose logs -f

# 代码更新后重新构建
docker-compose build --no-cache
docker-compose up -d

# 进入容器手动执行
docker-compose exec stock-analyzer python main.py --no-notify
```

### 代码质量检查
```bash
# 代码格式化
black .
isort .

# 代码检查
flake8 .

# 安全扫描
bandit -r . -x ./test_*.py
```

## 架构设计

### 程序入口与执行流程
- **main.py** - 主调度程序，协调整个分析流程
  - 解析命令行参数，选择执行模式
  - 初始化 StockAnalysisPipeline 并配置并发数
  - 管理完整分析流程：个股分析 → 大盘复盘 → 飞书文档生成
  - 支持三种模式：单次运行、定时任务、仅大盘复盘

### 配置系统（单例模式）
- **config.py** - 集中式配置管理，使用 dataclass
  - 从 .env 文件加载，回退到环境变量
  - 通过 `get_config()` 访问全局单例
  - 核心配置：API Keys、自选股列表、通知渠道、流控参数
  - `refresh_stock_list()` - 热重载自选股列表（定时任务模式下很有用）

### 数据层（策略模式）
- **data_provider/** - 多数据源自动切换机制
  - **base.py** - 定义 `BaseFetcher` 抽象基类和 `DataFetcherManager` 管理器
    - 所有数据源实现：`_fetch_raw_data()`、`_normalize_data()`
    - 管理器按优先级尝试（0=最高），失败自动切换下一个
    - 标准化列名：['date', 'open', 'high', 'low', 'close', 'volume', 'amount', 'pct_chg']
  - **efinance_fetcher.py** - 优先级 0（默认，最可靠）
  - **akshare_fetcher.py** - 优先级 1，同时提供实时行情和筹码分布
  - **tushare_fetcher.py** - 优先级 2，需要 token
  - **baostock_fetcher.py** - 优先级 3
  - **yfinance_fetcher.py** - 优先级 4，用于港股/美股
  - 每个数据源内置流控逻辑和随机延时，防止被封禁

### 存储层
- **storage.py** - SQLite 数据库抽象（`DatabaseManager`）
  - 表：daily_data（日线数据）、analysis_results（分析结果）
  - `has_today_data()` - 断点续传支持（跳过已获取的股票）
  - `get_analysis_context()` - 返回最近 N 天数据 + MA5/MA10/MA20 计算
  - 使用 SQLAlchemy ORM

### 分析流水线
1. **stock_analyzer.py** - 技术面趋势分析（均线排列、乖离率、量能）
   - `TrendAnalyzer.analyze()` 返回 `TrendAnalysisResult`
   - 检查 MA5 > MA10 > MA20（多头排列）、乖离率 < 5%（不追高）
   - 基于交易理念生成买入信号

2. **search_service.py** - 多源新闻情报搜索
   - 支持 Bocha、Tavily、SerpAPI，多 API Key 负载均衡
   - `search_comprehensive_intel()` - 三维度搜索：最新消息 + 风险排查 + 业绩预期
   - 返回格式化的情报报告供 AI 分析

3. **analyzer.py** - AI 分析引擎（Google Gemini）
   - `GeminiAnalyzer.analyze()` - 主入口，生成 `AnalysisResult`（决策仪表盘）
   - 综合：技术面数据 + 实时行情 + 筹码分布 + 趋势分析 + 新闻情报
   - 实现重试逻辑（429 限流处理，指数退避）
   - Gemini 不可用时自动切换到 OpenAI 兼容 API
   - 返回结构化仪表盘：综合评分、操作建议、买卖点位、检查清单

4. **market_analyzer.py** - 大盘复盘分析
   - `MarketAnalyzer.run_daily_review()` - 分析指数、板块表现、资金流向
   - 使用 AkShare 获取市场数据，搜索服务获取宏观新闻，AI 进行综合

### 通知层
- **notification.py** - 多渠道通知服务
  - 支持：企业微信、飞书、Telegram、邮件、Pushover、自定义 Webhook
  - `generate_dashboard_report()` - 生成决策仪表盘格式的报告
  - `generate_single_stock_report()` - 生成单股推送报告
  - 自动分割长消息以符合平台字节限制
  - 配置多个渠道时同时推送

- **feishu_doc.py** - 飞书云文档生成
  - `FeishuDocManager.create_daily_doc()` - 创建格式化的分析文档
  - 需要配置：FEISHU_APP_ID、FEISHU_APP_SECRET、FEISHU_FOLDER_TOKEN

### 定时调度
- **scheduler.py** - 基于 APScheduler 的定时任务封装
  - `run_with_schedule()` - 在配置的时间执行任务（如北京时间 18:00）
  - 由 `main.py --schedule` 模式调用

### WebUI（可选）
- **webui.py** - 简易 FastAPI 服务器，用于管理 .env 配置
  - 使用 `python main.py --webui` 启动
  - 访问 http://127.0.0.1:8000
  - 编辑自选股列表、查看/修改环境变量

## 核心设计模式

### 交易理念（内置）
系统内置保守的交易规则：
- **严禁追高**：乖离率 > 5% 时发出警告
- **趋势交易**：优先考虑 MA5 > MA10 > MA20 多头排列
- **回调买入**：寻找缩量回踩 MA5/MA10 支撑位的机会
- **精确点位**：AI 提供具体的买入价、止损价、目标价

### 防封禁策略
- **随机休眠**：每个数据获取器请求间加入 2-5 秒随机延时
- **低并发**：默认 max_workers=3（可通过 MAX_WORKERS 配置）
- **重试退避**：失败后指数级增加延时
- **多源切换**：某个数据源失败自动切换到下一个

### 通知策略
两种模式（通过 SINGLE_STOCK_NOTIFY 控制）：
1. **批量模式**（默认）：分析完所有股票 → 发送一份汇总仪表盘
2. **单股模式**：每分析完一只股票立即推送

### 断点续传
- `has_today_data()` 检查股票今日数据是否已获取
- 适用场景：部分失败后重跑、盘中新增股票

## 配置指南

### 必需的环境变量
```bash
# AI 模型（二选一）
GEMINI_API_KEY=your_key          # 推荐（有免费额度）
OPENAI_API_KEY=your_key          # 备选方案
OPENAI_BASE_URL=https://api.deepseek.com/v1  # OpenAI 兼容 API
OPENAI_MODEL=deepseek-chat

# 自选股列表
STOCK_LIST=600519,300750,002594  # 逗号分隔

# 至少配置一个通知渠道
WECHAT_WEBHOOK_URL=https://...
FEISHU_WEBHOOK_URL=https://...
TELEGRAM_BOT_TOKEN=...
TELEGRAM_CHAT_ID=...
EMAIL_SENDER=your@email.com
EMAIL_PASSWORD=auth_code
```

### 可选但推荐
```bash
# 新闻搜索（提升分析质量）
TAVILY_API_KEYS=key1,key2        # 支持多 Key 负载均衡
BOCHA_API_KEYS=key1,key2         # 中文搜索优化，带 AI 摘要
SERPAPI_API_KEYS=key1,key2

# 数据源
TUSHARE_TOKEN=your_token         # 获取高级数据

# 行为控制
SINGLE_STOCK_NOTIFY=true         # 每只股票分析完立即推送
MARKET_REVIEW_ENABLED=true       # 包含大盘复盘
SCHEDULE_TIME=18:00              # 每日执行时间（HH:MM）
```

## GitHub Actions 部署

系统设计为在 GitHub Actions 上零成本部署：

1. **Fork 仓库**并启用 Actions
2. **配置 Secrets**：Settings → Secrets and variables → Actions
   - 添加所有必需的环境变量作为 repository secrets
   - 包括自选股列表、API Keys、通知 Webhooks
3. **自动运行**：工作日北京时间 18:00 自动执行（cron: '0 10 * * 1-5'）
4. **手动触发**：Actions 标签页 → "每日股票分析" → Run workflow
5. **运行模式**：full（股票+大盘）、market-only（仅大盘）、stocks-only（仅股票）

工作流文件：`.github/workflows/daily_analysis.yml`

## 重要说明

### 股票代码格式
- A 股：6 位数字，无前缀（如 '600519'、'000001'、'300750'）
- H 股：使用腾讯代码格式，需要 yfinance 数据源

### 代理配置
国内访问 Gemini API 需要代理，在 main.py 中取消注释：
```python
os.environ["http_proxy"] = "http://127.0.0.1:10809"
os.environ["https_proxy"] = "http://127.0.0.1:10809"
```

### 新功能的文件结构
- 新数据源 → 在 data_provider/ 中实现 BaseFetcher
- 新通知渠道 → 在 notification.py 中添加到 NotificationChannel 枚举
- 新分析逻辑 → 扩展 analyzer.py 或创建独立分析器模块
- 新命令 → 在 main.py 的 parse_arguments() 中添加参数

### 日志系统
- 控制台 + 2 个日志文件：常规日志（INFO）+ 调试日志（DEBUG）
- 位置：./logs/ 目录
- 命名规则：stock_analysis_YYYYMMDD.log
- 自动轮转：10MB（常规日志）、50MB（调试日志）

### 本地测试
始终先使用 --no-notify 测试，避免误发通知：
```bash
python main.py --debug --stocks 600519 --no-notify
```

## 常见开发任务

### 添加新数据源
1. 在 data_provider/ 创建新文件
2. 继承 BaseFetcher
3. 实现 _fetch_raw_data() 和 _normalize_data()
4. 设置 priority（数字越小优先级越高）
5. 管理器自动发现并使用

### 修改 AI 提示词
- 编辑 analyzer.py 中的系统提示（GeminiAnalyzer 类）
- 仪表盘格式在提示模板中定义
- 注意 token 限制

### 添加新通知渠道
1. 在 NotificationChannel 枚举中添加新渠道
2. 在 NotificationService 中实现 send_to_X() 方法
3. 在 config.py 中添加配置变量
4. 更新 get_available_channels() 逻辑

