# 市场情报选品系统 - 技术方案

> **版本**: v1.0
> **更新时间**: 2026-04-16
> **状态**: 待开发

---

## 1. 系统架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              用户层                                      │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐ │
│  │   Vue 3 前端     │      │   REST API       │      │   WebSocket     │ │
│  │   (浏览器)       │      │   (HTTP/HTTPS)   │      │   (实时推送)     │ │
│  └────────┬────────┘      └────────┬────────┘      └────────┬────────┘ │
└───────────┼────────────────────────┼────────────────────────┼─────────┘
            │                        │                        │
            └────────────────────────┼────────────────────────┘
                                     │
┌─────────────────────────────────────▼─────────────────────────────────┐
│                            Spring Boot 应用层                            │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Controller 层                                │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │   │
│  │  │ AnalysisCtl │  │ MonitorCtl  │  │ ReportCtl            │  │   │
│  │  │ 选品分析接口  │  │ 监控任务接口  │  │ 报告管理接口          │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│  ┌─────────────────────────────────▼─────────────────────────────────┐ │
│  │                      Service 层                                    │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │ │
│  │  │ AnalysisSvc  │  │ MonitorSvc   │  │ ReportSvc             │  │ │
│  │  │ 选品分析逻辑  │  │ 监控调度逻辑  │  │ 报告生成逻辑          │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │ │
│  │  │ SourceManager│  │ AIFacade     │  │ SupplierFinder       │  │ │
│  │  │ 数据源管理器  │  │ AI 接口封装   │  │ 货源匹配服务          │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                    │                                      │
│  ┌─────────────────────────────────▼─────────────────────────────────┐ │
│  │                   Source Adapter 层 (插件化)                        │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │ │
│  │  │ AmazonAdapter│  │ RedditAdapter│  │ 1688Adapter          │  │ │
│  │  │              │  │              │  │                      │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │ │
│  │                          ▲                                        │ │
│  │                          │ 可扩展接口                             │ │
│  │  ┌──────────────┐       │       ┌──────────────────────┐       │ │
│  │  │ TwitterAdapter│ ─────┴────── │ FutureAdapter        │       │ │
│  │  │ (未来扩展)    │               │  (预留扩展位)         │       │ │
│  │  └──────────────┘               └──────────────────────┘       │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
                                     │
┌─────────────────────────────────────▼─────────────────────────────────┐
│                          基础设施层                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │   MySQL 8         │  │   OpenCLI CLI     │  │   Qwen API           │  │
│  │   数据持久化        │  │   爬虫执行引擎     │  │   AI 分析             │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 2. 模块职责边界

### 2.1 Controller 层

| 模块 | 职责 | 公开接口 |
|------|------|----------|
| **AnalysisController** | 选品分析入口 | `POST /api/analysis/start` |
| **MonitorController** | 监控任务 CRUD | `GET/POST/PUT/DELETE /api/monitors` |
| **ReportController** | 报告管理 | `GET /api/reports`, `GET /api/reports/{id}/download` |
| **SourceController** | 数据源状态 | `GET /api/sources/status` |

### 2.2 Service 层

| 模块 | 职责 | 核心方法 |
|------|------|----------|
| **AnalysisService** | 编排分析流程 | `runAnalysis(query)` |
| **MonitorService** | 监控任务管理 | `create/update/delete/trigger` |
| **ReportService** | 报告生成存储 | `generate/download/list` |
| **SourceManager** | 数据源统一管理 | `getAdapter(type) / scrapeAll()` |
| **AIFacade** | AI 接口封装 | `analyze(data, type)` |
| **SupplierFinder** | 货源搜索匹配 | `findSuppliers(trends)` |

### 2.3 Source Adapter 层

| 适配器 | 职责 | 实现接口 |
|--------|------|----------|
| **AmazonAdapter** | 爬取 BS 榜单 | `SourceAdapter` |
| **RedditAdapter** | 爬取帖子/评论 | `SourceAdapter` |
| **1688Adapter** | 搜索货源 | `SupplierAdapter` |
| **TwitterAdapter** | (预留) | `SourceAdapter` |

---

## 3. 核心流程

### 3.1 选品分析流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      选品分析流程                                  │
└─────────────────────────────────────────────────────────────────┘

用户输入: "收纳盒"

        │
        ▼
┌───────────────────┐
│ 1. 参数校验       │
│    - 关键词非空    │
│    - 长度限制     │
└────────┬──────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────┐
│ 2. 并行爬取多源数据                                          │
│                                                           │
│   ┌─────────────┐  ┌─────────────┐                         │
│   │ Amazon BSR │  │ Reddit 帖子 │                         │
│   │  (OpenCLI) │  │  (OpenCLI) │                         │
│   └──────┬──────┘  └──────┬──────┘                         │
│          │                │                                │
└─────────┼────────────────┼────────────────────────────────┘
          │                │
          └────────┬───────┘
                   ▼
         ┌─────────────────┐
         │ 3. 数据归一化    │
         │    - 统一格式    │
         │    - 去重       │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 4. 构造 AI Prompt│
         │    - 趋势分析    │
         │    - 选品建议    │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 5. 调用 Qwen    │
         │    - 分析趋势    │
         │    - 生成建议    │
         └────────┬────────┘
                  │
                  ▼
         ┌───────────────────────────────────────────┐
         │ 6. 从建议中提取产品关键词                    │
         │    e.g. "可折叠收纳箱" → 搜索 1688        │
         └───────────────────────┬───────────────────┘
                                 │
                                 ▼
         ┌───────────────────────────────────────────┐
         │ 7. 搜索 1688 货源 (并行)                    │
         │    - 供应商列表     - 价格                 │
         │    - MOQ           - 评分                 │
         └───────────────────────┬───────────────────┘
                                 │
                                 ▼
         ┌───────────────────────────────────────────┐
         │ 8. 计算利润估算                            │
         │    - 利润率 = (Amazon价×汇率 - 1688价)/Amazon价 │
         │    - 去除运费和平台费                      │
         └───────────────────────┬───────────────────┘
                                 │
                                 ▼
         ┌───────────────────────────────────────────┐
         │ 9. 生成报告                                │
         │    - Markdown 格式  - PDF 格式           │
         │    - 存入数据库     - 返回给前端           │
         └───────────────────────────────────────────┘
```

### 3.2 竞品监控流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      竞品监控流程                                  │
└─────────────────────────────────────────────────────────────────┘

定时触发 (Cron) / 手动触发

        │
        ▼
┌───────────────────┐
│ 获取监控任务列表   │
│ - Amazon ASIN     │
│ - 监控频率        │
└────────┬──────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────┐
│ 遍历每个监控任务                                              │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 爬取 Amazon 商品详情 (OpenCLI)                      │    │
│  │   - 当前价格                                        │    │
│  │   - 评论数                                          │    │
│  │   - 评分                                            │    │
│  │   - BS 排名                                        │    │
│  └───────────────────────┬─────────────────────────────┘    │
│                          ▼                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 与上次数据对比                                       │    │
│  │   - 价格变动?  - 评论增长?  - 评分变动?             │    │
│  └───────────────────────┬─────────────────────────────┘    │
│                          ▼                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 写入价格历史                                        │    │
│  └───────────────────────┬─────────────────────────────┘    │
│                          ▼                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 检测到变化? → 生成竞品动态报告                      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
└───────────────────────────────────────────────────────────┘
```

---

## 4. 数据模型

### 4.1 核心实体

```
┌─────────────────────────────────────────────────────────────────┐
│                          实体关系图                               │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   Monitor    │       │   Product    │       │    Report    │
│   监控任务    │       │   产品/竞品   │       │   分析报告    │
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id           │       │ id           │       │ id           │
│ name         │       │ external_id   │       │ type         │
│ source_type  │──────▶│ source_type  │       │ query        │
│ target_id    │       │ title        │       │ summary       │
│ schedule     │       │ url          │       │ content_md    │
│ enabled      │       │ current_price│       │ content_pdf   │
│ created_at   │       │ currency     │       │ created_at    │
└──────────────┘       │ created_at   │       └──────────────┘
                       │ updated_at   │              │
                       └──────┬───────┘              │
                              │                       │
                              │ 1:N                  │ N:1
                              ▼                       │
                    ┌──────────────┐                  │
                    │ PriceHistory│                  │
                    │ 价格历史      │                  │
                    ├──────────────┤                  │
                    │ id           │                  │
                    │ product_id   │◀─────────────────┘
                    │ price        │
                    │ recorded_at  │
                    └──────────────┘


┌──────────────┐       ┌──────────────┐
│  AIInsight   │       │  Supplier    │
│  AI 分析洞察  │       │   1688 货源  │
├──────────────┤       ├──────────────┤
│ id           │       │ id           │
│ report_id    │       │ external_id │
│ product_name │       │ title       │
│ trend_score   │       │ price       │
│ competition   │       │ moq         │
│ advice       │       │ seller_name  │
│ created_at   │       │ location     │
└──────────────┘       │ rating       │
                       │ product_url  │
                       └──────────────┘
```

### 4.2 主要表结构

| 表名 | 用途 | 关键字段 |
|------|------|----------|
| `monitors` | 监控任务 | source_type, target_id, schedule, enabled |
| `products` | 产品/竞品 | external_id, source_type, title, url, current_price |
| `price_history` | 价格历史 | product_id, price, recorded_at |
| `reports` | 分析报告 | type, query, summary, content_md, content_pdf |
| `ai_insights` | AI 洞察 | report_id, product_name, trend_score, advice |
| `suppliers` | 1688 货源 | external_id, title, price, moq, seller_name |

---

## 5. API 设计

### 5.1 选品分析

```
POST /api/analysis/start
Body: { "query": "收纳盒", "sources": ["amazon", "reddit"] }
Response: { "reportId": "rpt_xxx", "status": "PROCESSING" }

GET /api/analysis/{reportId}/status
Response: { "status": "COMPLETED", "progress": 100, "report": {...} }
```

### 5.2 竞品监控

```
GET  /api/monitors           → 列出所有监控任务
POST /api/monitors           → 创建监控任务
     Body: { "name": "竞品A", "sourceType": "AMAZON", "targetId": "B0XXXXX", "schedule": "0 0 * * *" }
PUT  /api/monitors/{id}      → 更新监控任务
DELETE /api/monitors/{id}    → 删除监控任务
POST /api/monitors/{id}/trigger → 手动触发一次
```

### 5.3 报告管理

```
GET  /api/reports                    → 列出报告 (分页)
GET  /api/reports/{id}               → 获取报告详情
GET  /api/reports/{id}/download      → 下载 PDF
GET  /api/reports/{id}/products      → 报告中涉及的产品
```

### 5.4 数据源状态

```
GET /api/sources/status               → 各数据源连接状态
GET /api/sources/amazon/categories   → Amazon 品类列表
```

---

## 6. 扩展机制：适配器模式

### 6.1 SourceAdapter 接口

```java
public interface SourceAdapter {

    // 数据源标识
    String getName();

    // 数据源类型
    SourceType getType();  // SALES_RANK, SOCIAL, SEARCH_TREND

    // 爬取数据
    List<RawData> scrape(ScrapeRequest request);

    // 数据归一化
    NormalizedItem normalize(RawData raw);
}
```

### 6.2 适配器注册机制

```java
@Service
public class SourceManager {

    private final Map<String, SourceAdapter> adapters = new HashMap<>();

    // Spring 自动注入所有实现 SourceAdapter 的 Bean
    @Autowired
    public void registerAdapters(List<SourceAdapter> adapterList) {
        for (SourceAdapter adapter : adapterList) {
            adapters.put(adapter.getName(), adapter);
        }
    }

    public SourceAdapter getAdapter(String name) {
        return adapters.get(name);
    }
}
```

### 6.3 新增数据源流程

```
新增 Twitter 爬虫：

1. 创建 TwitterAdapter 实现 SourceAdapter
2. @Component 注解，自动注册到 SourceManager
3. 实现 scrape() 和 normalize() 方法
4. 前端即可选择 "twitter" 数据源

改动范围：仅限新增文件，不触动核心逻辑
```

---

## 7. AI 接口封装

### 7.1 AIFacade 设计

```java
@Service
public class AIFacade {

    // 支持多 Provider
    private final Map<String, LLMProvider> providers;

    // AI 分析入口
    public AnalysisResult analyze(AnalysisInput input, String provider) {
        LLMProvider llm = providers.get(provider);

        // 构造 Prompt
        String prompt = buildPrompt(input);

        // 调用 LLM
        String response = llm.complete(prompt);

        // 解析响应
        return parseResponse(response);
    }
}
```

### 7.2 Prompt 模板

```java
public class PromptTemplates {

    // 趋势分析 Prompt
    public static String TREND_ANALYSIS = """
        你是一位跨境电商选品专家。分析以下来自多个平台的数据，发现趋势和机会。

        【Amazon 热销榜】
        {amazon_data}

        【Reddit 社区讨论】
        {reddit_data}

        请分析并返回：
        1. 热销产品的共同特征
        2. 社交媒体上的新兴趋势
        3. 3-5 个高潜力选品建议（含理由）
        4. 每个建议的风险提示

        返回格式：JSON
        {
          "trends": [...],
          "topProducts": [...],
          "risks": [...]
        }
        """;
}
```

---

## 8. 调度机制

### 8.1 Scheduler 设计

```java
@Service
public class MonitorScheduler {

    // 所有启用的监控任务
    @Scheduled(cron = "0 */5 * * * *")  // 每 5 分钟检查一次
    public void checkMonitors() {
        List<Monitor> dueMonitors = monitorService.getDueMonitors();

        for (Monitor monitor : dueMonitors) {
            monitorService.execute(monitor);
        }
    }
}
```

### 8.2 任务执行流程

```
监控任务触发
    │
    ▼
检查是否到执行时间
    │
    ▼
获取数据源适配器
    │
    ▼
调用 OpenCLI 爬取
    │
    ▼
对比上次数据
    │
    ├── 有变化 → 写入历史 + 生成报告
    │
    └── 无变化 → 仅更新 last_run 时间
```

---

## 9. OpenCLI 集成

### 9.1 OpenCLI 调用方式

```java
@Service
public class OpenCLIService {

    public String execute(String... args) {
        ProcessBuilder pb = new ProcessBuilder("opencli", args);
        pb.redirectErrorStream(true);

        Process process = pb.start();

        // 读取输出
        String output = new String(process.getInputStream().readAllBytes());

        // 等待完成
        int exitCode = process.waitFor();

        if (exitCode != 0) {
            throw new OpenCLIException("OpenCLI failed: " + output);
        }

        return output;
    }

    // 示例调用
    public List<Product> searchAmazon(String query, int limit) {
        String output = execute("amazon", "search", query, "--limit", String.valueOf(limit), "--format", "json");
        return parseJson(output);
    }

    public List<Post> searchReddit(String query, int limit) {
        String output = execute("reddit", "search", query, "--limit", String.valueOf(limit), "--format", "json");
        return parseJson(output);
    }

    public List<Supplier> search1688(String query, int limit) {
        String output = execute("1688", "search", query, "--limit", String.valueOf(limit), "--format", "json");
        return parseJson(output);
    }
}
```

---

## 10. 部署架构

### 10.1 Docker Compose 部署

```yaml
services:

  # MySQL 数据库
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: market_monitor
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping"]

  # Spring Boot 应用
  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DB_HOST: mysql
      DB_PORT: 3306
      DB_NAME: market_monitor
      DB_USER: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
      QWEN_API_KEY: ${QWEN_API_KEY}
    depends_on:
      mysql:
        condition: service_healthy
    ports:
      - "8080:8080"
    volumes:
      - ./reports:/app/reports  # 报告存储

volumes:
  mysql_data:
```

### 10.2 一键部署命令

```bash
# 1. 复制环境配置
cp .env.example .env
# 编辑 .env，设置 MySQL 密码和 Qwen API Key

# 2. 启动所有服务
docker-compose up -d --build

# 3. 初始化数据库
docker-compose exec app ./mvnw flyway:migrate

# 4. 访问系统
# 前端: http://localhost:8080
# API:  http://localhost:8080/api
```

---

## 11. 安全设计

| 层面 | 措施 |
|------|------|
| **API 认证** | API Key 认证（后续扩展 JWT） |
| **密钥存储** | API Key 存环境变量，不写代码 |
| **数据库** | MySQL 密码强密码，限制远程访问 |
| **爬虫隔离** | 每个租户独立 Chrome Profile |
| **AI 调用** | Qwen API Key 仅服务端使用，不暴露前端 |

---

## 12. 性能优化

| 优化点 | 策略 |
|--------|------|
| **多源并行爬取** | CompletableFuture 并行调用 OpenCLI |
| **结果缓存** | Redis 缓存 1688 搜索结果（1小时） |
| **异步报告生成** | 报告生成异步执行，前端轮询状态 |
| **分页加载** | 报告列表默认 20 条，支持游标分页 |
| **数据库索引** | source_type, external_id, created_at 索引 |

---

## 13. 监控系统

| 指标 | 监控方式 |
|------|----------|
| **服务可用性** | Spring Boot Actuator + Docker Health |
| **API 响应时间** | Micrometer Metrics |
| **定时任务执行** | 记录执行日志，失败告警 |
| **数据源状态** | OpenCLI 调用成功率 |
| **AI API 调用** | Qwen API 响应时间和错误率 |

---

## 14. 技术栈汇总

| 组件 | 技术选型 |
|------|----------|
| 后端框架 | Spring Boot 3.x (Java 21) |
| 前端框架 | Vue 3 + Element Plus |
| 数据库 | MySQL 8 (Docker) |
| 爬虫引擎 | OpenCLI CLI (ProcessBuilder) |
| AI 接口 | 通义千问 (Qwen) API |
| 报告导出 | iText (PDF) |
| 定时调度 | Spring @Scheduled |
| 部署方式 | Docker Compose |

---

## 15. 文件结构

```
market-intelligence/
├── src/main/java/com/monitor/
│   ├── MarketIntelligenceApplication.java
│   │
│   ├── controller/
│   │   ├── AnalysisController.java
│   │   ├── MonitorController.java
│   │   ├── ReportController.java
│   │   └── SourceController.java
│   │
│   ├── service/
│   │   ├── AnalysisService.java
│   │   ├── MonitorService.java
│   │   ├── ReportService.java
│   │   ├── SourceManager.java
│   │   ├── AIFacade.java
│   │   ├── OpenCLIService.java
│   │   └── SupplierFinder.java
│   │
│   ├── adapter/
│   │   ├── SourceAdapter.java
│   │   ├── AmazonAdapter.java
│   │   ├── RedditAdapter.java
│   │   └── 1688Adapter.java
│   │
│   ├── entity/
│   │   ├── Monitor.java
│   │   ├── Product.java
│   │   ├── PriceHistory.java
│   │   ├── Report.java
│   │   ├── AIInsight.java
│   │   └── Supplier.java
│   │
│   ├── repository/
│   │   ├── MonitorRepository.java
│   │   ├── ProductRepository.java
│   │   ├── PriceHistoryRepository.java
│   │   ├── ReportRepository.java
│   │   └── SupplierRepository.java
│   │
│   ├── dto/
│   │   ├── AnalysisRequest.java
│   │   ├── AnalysisResponse.java
│   │   ├── MonitorRequest.java
│   │   └── ReportResponse.java
│   │
│   ├── config/
│   │   ├── AIConfig.java
│   │   ├── OpenCLIConfig.java
│   │   └── SchedulerConfig.java
│   │
│   └── util/
│       ├── PromptTemplates.java
│       └── ReportGenerator.java
│
├── src/main/resources/
│   ├── application.yml
│   ├── templates/           # Thymeleaf 模板
│   └── db/migration/        # Flyway 迁移脚本
│
├── src/test/java/
│   └── com/monitor/
│       ├── service/
│       └── adapter/
│
├── docker/
│   └── Dockerfile
│
├── docker-compose.yml
├── .env.example
├── pom.xml
└── README.md
```
