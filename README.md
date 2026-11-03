<p align="center">
  <sub>Spring Boot 3 · WebFlux · R2DBC · Netty · Vue 3 · LangChain4j</sub>
</p>

# 商城 + 实时服务监控

一个能跑通的电商系统，和一套自己做的主机 / 服务监控。

商品可以浏览、加购、下单，可以在对话框里用自然语言让「导购」帮你挑；服务端由 Netty 手写，机器上跑一个采集端，把 CPU、内存、磁盘和心跳上报到监控台。

---

## 里面有什么（6 个模块）

| 模块 | 技术 | 职责 |
|---|---|---|
| `ecommerce-backend` | Spring Boot 3.3 · **WebFlux** · **R2DBC** · LangChain4j | 商品 / 购物车 / 订单 / 聊天导购 / 支付回调 |
| `ecommerce-front` | Vue 3 + TypeScript | 商品页、购物车、聊天对话框 |
| `monitor-server` | **Netty**（TCP + HTTP + WebSocket）· OSHI | 节点注册、心跳维持、离线判定、主机指标采集 |
| `monitor-front` | Vue 3 | 节点在线状态、指标仪表盘 |
| `data-crawler` | Python | 商品采集、清洗、向量化入库 |
| `docs` | — | 需求文档、前后端联调说明、知识库设计 |

---

## 架构

```text
                     ┌──────────────────────────┐
  浏览器 ──HTTP/WS──▶ │  ecommerce-backend       │
                     │  WebFlux + R2DBC(非阻塞)  │──▶ PostgreSQL
                     └──────────┬───────────────┘
                                │ LangChain4j
                                ▼
                          DeepSeek（RAG 导购）

   采集端 ──TCP(自定义协议)──▶ ┌──────────────────────┐
   浏览器 ──HTTP/WebSocket──▶ │  monitor-server       │──▶ 历史库(JDBC)
                              │  Netty · 心跳 · 注册表  │
                              └──────────────────────┘
```

---

## 我遇到的问题，以及怎么解的

> 这一节是重点。下面每一条都是开发时真的卡过的地方。

### 1. 怎么判断一个节点是「真挂了」，还是只是网络抖了一下

**问题**：监控最怕误报。一次超时就判离线，会导致告警风暴；等太久，真挂了又没人知道。

**做法**：采集端定时上报心跳，服务端维护一张注册表（`MonitorRegistry`），节点状态只有 `ONLINE / OFFLINE` 两态，由「最后一次心跳时间 + 超时阈值」驱动状态迁移，而不是靠连接断开——连接断开是弱信号，进程假死时 TCP 还连着。

**边界**：当前没有做多级阈值（例如「可疑 → 离线」），节点多的时候会有误判，这是我明确知道还没解决的地方。

### 2. 同一个服务要同时接机器和浏览器，协议不一样

**问题**：采集端是长连接上报，浏览器要的是 HTTP 和 WebSocket，一套端口进去要能分清是谁。

**做法**：自定义了一套 `MonitorProtocol` 定帧格式，TCP 和 HTTP/WebSocket 各自独立的 handler（`MonitorTcpHandler` / `MonitorHttpWebSocketHandler`），但共享同一个注册表——协议解析分开，状态收敛到一处。

### 3. 响应式栈里最怕「混进一个阻塞调用」

**问题**：后端用了 WebFlux + R2DBC，全链路 non-blocking。这种架构的好处是省线程，坏处是——**只要有一个地方阻塞了事件循环，整个服务都会卡住**，而且不报错，只是变慢。

**做法**：业务主链路（商品、购物车、聊天）严格走 R2DBC；监控侧的历史落库是 JDBC，天然阻塞，所以放在 `monitor-server` 独立进程里（`JdbcHistoryStore`），不进入 Web 的事件循环。

**学到的东西**：响应式不是「换个依赖」，而是**整条链路上不能有任何一个人偷偷阻塞**。

### 4. 知识库太粗，导购答得像百科全书

**问题**：最初耳机品类只有 3 条知识（通勤 / 佩戴 / 音质），AI 每次都给一段"泛泛而谈"，落不到具体商品上。

**做法**：把知识库按维度拆到 15 条，覆盖场景、降噪、续航、通话、音质、延迟、佩戴、防水、蓝牙、品牌、常见痛点等；每条挂 3–6 个统一词表的标签，方便向量检索交叉命中。

同时把对话流程从 4 步扩成「预算 4 档 → 场景 4 类 → 功能 6 维（可多选）→ 外观 4 类 + 专项确认」，让每一轮都在收敛，而不是重复提问。

**为什么拆细有用**：检索命中的是"一条具体判断"，而不是"一段品类介绍"，模型才有东西可引用。

### 5. 商品数据要自己采、自己洗、自己向量化

**问题**：没有现成商品库，价格和参数还经常变。

**做法**：Python 采集脚本抓取 → 清洗成统一字段 → `knowledge-vectorizer.py` 批量向量化 → 入 PostgreSQL。向量化和商品入库是两条流水线，互不阻塞。

### 6. 支付回调不能"收到就算成功"

**问题**：支付宝沙箱回调可能重复投递，也可能被伪造。

**做法**：回调先验签，再按订单号做幂等处理（已处理过的直接返回成功），订单状态用状态机推进，不允许从"已支付"退回"待支付"。

---

## 技术栈

| 层 | 选型 |
|---|---|
| 后端 | Spring Boot 3.3 · Java 17 · **WebFlux** · **R2DBC** · Validation |
| AI | **LangChain4j** · DeepSeek · RAG（知识库 + 向量检索） |
| 监控服务 | **Netty**（TCP / HTTP / WebSocket）· **OSHI** 主机指标 |
| 前端 | Vue 3 · TypeScript · Vite · Composition API |
| 数据 | PostgreSQL（业务）· PostgreSQL（监控历史） |
| 工具链 | Maven · PowerShell 启动脚本 · Python 数据脚本 |

---

## 快速开始

```bash
# 1. 初始化数据库
psql -U postgres -f ecommerce-backend/src/main/resources/schema.sql

# 2. 启动电商后端
./start-ecommerce-backend.ps1

# 3. 启动监控服务
./start-monitor-server.ps1

# 4. 前端
cd ecommerce-front && npm install && npm run dev
```

配置文件见 `ecommerce-backend/src/main/resources/application.properties`，需要填数据库连接与 DeepSeek API Key。

---

## 已知限制

- 这是个人练手项目，**没有上线运营**，也没有真实用户
- 监控只做了单机指标的采集与展示，没有做集群聚合
- 离线判定是单一阈值，节点多时会有误判（见问题 1）
- 推荐结果没有做过评测集，所以这里不写任何准确率数字

---

## 文档

- [需求文档](docs/requirements.md)
- [前后端联调说明](docs/frontend-backend-integration.md)
- [项目结构说明](PROJECT_STRUCTURE.md)
- [知识库升级记录](HEADPHONE_KB_UPGRADE_REPORT.md)

---

## License

MIT
