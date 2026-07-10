# 项目展示

<div style="display: flex; gap: 10px;">
  <img src="https://free.picui.cn/free/2026/03/29/69c8db2c32d74.png" width="800" />
  <img src="https://free.picui.cn/free/2026/03/29/69c8db2ead009.png" width="800" />
  <img src="https://free.picui.cn/free/2026/03/29/69c8db2b93e32.png" width="800" />
  <img src="https://free.picui.cn/free/2026/03/29/69c8db2b89818.png" width="800" />
  <img src="https://free.picui.cn/free/2026/03/29/69c8db2bd181e.png" width="800" />
  <img src="https://free.picui.cn/free/2026/03/29/69c8db2c30a4b.png" width="800" />
</div>

# 知光平台-知识获取与分享社区

- **项目地址**：https://github.com/WaitMeBuyJuice/KnowLight
- **项目概述**：该平台面向用户社交和知识交流的社区场景，具备用户认证、知文分享、Feed 流推荐、智能搜索等功能；结合AI实现LLM知文总结、RAG知识库问答，提升用户体验；并针对 C 端高并发读、突发热点等场景，进行多级缓存、事件分发、并发控制等多维度优化，保障多用户在线实时创作与互动。
- **技术栈**：Spring Boot、Spring Security、MyBatis、MySQL、Elasticsearch、Redis、Caffeine、Canal、Kafka、MinIO、RAG 
- **项目细节与亮点**：
  - **认证系统**：基于 Spring Security 开发 JWT 双令牌认证系统，通过分布式锁 + Redis 黑白名单，支持即时令牌撤销，兼顾高并发安全与高性能。
  - **发布系统**：编排基于知文状态机的渐进式发布流程，以预签名+前端直传的形式，上传知文内容文件至 MinIO ，并接入 Qwen3.7plus AI 一键生成文章摘要。
  - **Feed流**：基于 Redis + Caffeine 构建Feed流四级缓存架构，以游标分页保证深分页稳定性；针对高并发场景，设计hotkey探测机制，结合随机TTL抖动抗缓存击穿雪崩，并由分布式锁避免并发回源风暴。
  - **搜索系统**：基于 Elasticsearch 构建知文搜索系统，具备低延迟前缀联想、标签过滤，内容高亮等功能，通过构造ES查询规则，融合 BM25 相关性与点赞等业务权重优化搜索排序。
  - **计数系统**：以 Redis SDS 结构存储知文/用户计数，采用 Hash 聚合计数事件 + 定时Lua脚本批量汇总 的两阶段更新策略，并作异常重建、降低兜底，应对高并发场景，QPS达8K+。
  - **事件分发架构**：采用 outbox + Canal + Kafka 模式实现业务主链路与下游服务解耦，链接用户关系落库、ES索引增量同步、计数更新等下游服务，并做幂等性、顺序消费等处理，提高业务接口相应速度与QPS。
  - **AI 问答系统**：开发知光平台 RAG 知识问答系统，通过合理分块、幂等删除保持单一版本、预索引减少首次提问等待时间等，显著提升用户围绕单篇知文的智能问答效率与准确性。

---

# 项目优化点

## 1. 认证鉴权：单令牌 → 双令牌

**优化前**
采用单一长有效期 accessToken（如 7 天）做鉴权，无 refreshToken 机制。

**优化原因**

- 长 token 泄露窗口大，且 JWT 无状态签发后无法主动撤销，攻击者拿到可一直用到过期。
- 若改短 token（如 15min）则用户频繁被强制重新登录，体验差。
- 无法实现「登出即时生效」「改密/封号强制下线」等能力。

**优化后**
采用 accessToken（15min，无状态）+ refreshToken（7 天，有状态 Redis 白名单）双令牌：

- accessToken 短命，仅用于接口鉴权，靠 JWT 内 exp 字段自然过期。
- refreshToken 长命但低频，仅用于刷新接口；jti 写入 Redis 白名单 `auth:rt:{userId}:{tokenId}`，支持主动撤销。
- 刷新时执行 rotation：旧 refreshTokenId 立即失效，签发新令牌对，防止旧 token 被盗用。

**收益**

- accessToken 泄露窗口压缩到 15min。
- refreshToken 可主动撤销（登出 `revokeToken`、改密/封号 `revokeAll`），实现即时失效。
- 用户无感续期，无需频繁登录。

**代价**

- 双令牌管理复杂度增加（签发、刷新、撤销、rotation）。
- refreshToken 白名单需 Redis 存储，多一次 IO。
- 刷新接口本身带来额外请求开销。

---

## 2. 关注/取关 & 点赞/收藏接口限流：加入令牌桶

**优化前**
关注/取关已有限流，但点赞/收藏接口无限流；脚本可对点赞接口每秒上千次调用。

**优化原因**

- 位图幂等挡住了「重复点赞重复加计数」，但挡不住「点赞→取消→点赞→取消」循环——每次状态变化都发 Kafka 事件，刷爆聚合桶与计数链路。
- 恶意流量高频调用会打爆 Redis 位图操作（单 key 串行）和 Kafka 写入。
- 无限流时热点知文易被刷出虚假活跃度，污染 ES 冗余计数与推荐排序。

**优化后**
对点赞/收藏/关注/取关接口统一加用户维度令牌桶限流（复用关注模块已有的 Lua 令牌桶脚本）：

- key：`rl:like:{userId}` / `rl:fav:{userId}` / `rl:follow:{userId}`
- 参数：容量 10、每秒补 1，突发可连发 10 次，之后每秒限 1 次。

**收益**

- 挡住脚本刷接口，保护 Redis 位图与 Kafka 不被恶意流量打爆。
- 正常用户无感知（极少一秒内点赞超 10 次）。
- 复用现成 Lua 脚本，改造成本低。

**代价**

- 每次请求多一次 Redis 令牌桶查询。
- 限流参数（容量/速率）需按线上数据调优，误伤极端高频用户。
- 令牌桶 key 随用户增长，需关注 Redis 内存占用。

---

## 3. 对象存储中间件：阿里云 OSS → MinIO

**优化前**
知文正文、图片等文件存阿里云 OSS，按存储量与请求次数计费。

**优化原因**

- OSS 按存储量 + 请求次数收费，随内容量增长成本持续上升。
- 数据托管在云厂商，不自主可控，私有化部署与数据合规场景受限。

**优化后**
改用自建 MinIO（S3 兼容协议）：

- 预签名直传逻辑不变，仅替换 endpoint 与访问凭证。
- MinIO 部署在自有服务器，数据落本地磁盘。

**收益**

- 成本可控：服务器一次性投入，不随请求次数叠加费用。
- 数据自主可控：满足私有化部署与合规要求。
- S3 兼容协议，未来可平滑切回 OSS 或其他 S3 兼容存储，避免厂商锁定。

**代价**

- 需自行运维 MinIO 集群（部署、监控、备份、扩容、磁盘故障处理）。
- 无云厂商 SLA 保障，宕机与数据持久性需自担风险。
- 高可用与灾备需自行设计（多副本、纠删码、异地备份）。

---

## 4. 三类并发场景加分布式锁：令牌刷新 / 页面缓存重建 / SDS 计数重建

**优化前**
三处存在并发隐患：

- **令牌刷新**：同一 refreshToken 并发刷新，产生多对新令牌，旧 jti 被多次撤销、新白名单记录错乱。
- **Feed 页面缓存失效重建**：缓存 miss 时多请求并发回源 MySQL，打爆 DB。
- **SDS 计数缺失重建**：多请求并发触发 BITCOUNT 扫描位图分片，打爆 Redis。

**优化原因**

- 并发重复执行导致数据错乱（令牌多对、计数被多次写回覆盖）。
- 重建类操作开销大（BITCOUNT、回源 DB），并发触发形成重建风暴。

**优化后**
三处分别加 Redis 分布式锁（SET NX PX）：

- 令牌刷新：`lock:refresh:{tokenId}`，串行化 rotation，同一 refreshToken 同时只有一个刷新生效。
- Feed 页面：`lock:feed:page:{key}` single-flight，单飞回源，其余请求等待复用结果。
- SDS 重建：`lock:sds-rebuild:{etype}:{eid}`，单实体单重建，其余请求快速失败返回默认值。
- 配合 SDS 重建的 Redisson RRateLimiter 做全局重建限流，双重防风暴。

**收益**

- 避免并发重复执行，保证令牌 rotation、页面回源、计数重建的数据一致性。
- 防止重建风暴打爆 Redis/DB。
- 单飞复用：Feed 页面并发请求只回源一次，其余等结果。

**代价**

- 每次请求多一次锁查询（抢锁/释放）。
- 抢锁失败请求需等待或快速失败，极端场景下短暂返回默认值。
- 锁 TTL 需调优：过短导致重建未完成锁释放，过长导致崩溃后死锁。

---

## 5. Feed 流分页：offset → cursor 游标分页

**优化前**
Feed 流采用 page/size 偏移分页，深分页时 MySQL `LIMIT offset, size` 需扫描跳过 offset 行，性能随 offset 增大而劣化。

**优化原因**

- offset 大时 MySQL 要扫描并丢弃大量行，IO 与 CPU 浪费。
- Feed 流虽主要翻前几页，但深翻场景（如翻历史）性能不可控。
- 偏移分页在数据动态插入时还会出现重复/漏数据。

**优化后**
改为 cursor 游标分页，cursor = 上一页最后一条的 `publish_time + content_id`：

- 查询：`WHERE publish_time < ? OR (publish_time = ? AND content_id < ?) ORDER BY publish_time DESC, content_id DESC LIMIT ?`
- 走 (publish_time, content_id) 复合索引范围扫描，深分页性能恒定。
- cursor 经 Base64URL 编码后返回前端，翻页时原样带回。

**收益**

- 深分页性能恒定，不随页码增大而劣化。
- 走索引范围查询，避免 offset 扫描丢弃。
- 数据动态插入时游标定位稳定，不会重复/漏数据。

**代价**

- 不能随机跳页，只能下一页（影响「跳到第 N 页」交互）。
- cursor 需编码传前端，增加前后端约定。
- 排序字段必须包含唯一键（content_id 兜底）保证游标稳定。

---

## 6. Feed 流缓存结构：三层 → 四层，新增 L1 Caffeine 热门知文详情

**优化前**
Feed 流三层缓存：

- L2 = Caffeine 整页（FeedItemResponse 列表）
- L1 = Redis 页面骨架
- L0 = Redis 片段（`feed:item:{id}` 单篇知文）

**优化原因**

- 热门知文详情每次仍需穿透到 L0 Redis 片段读取，热 key 频繁访问 Redis，网络 RTT 累积。
- L2 Caffeine 缓存所有页（含深页、冷页），收益低的页面频繁重建浪费本地内存。

**优化后**
扩展为四层缓存：

- **L3 = Caffeine Feed 流整页**（原 L2，缩小范围：仅缓存首页 + hotkey 达标页，减少收益低页面的重建开销）
- **L2 = Redis Feed 流页面骨架**（原 L1）
- **L1 = Caffeine 整片知文**（新增，缓存热门知文详情，命中后无需穿透 Redis）
- **L0 = Redis 知文片段**（原 L0）

**收益**

- 热门知文详情命中 L1 Caffeine，省去 Redis 网络 RTT，本地内存读取更快。
- L3 缩小范围只缓存首页与 hotkey 达标页，降低冷页重建开销，本地内存利用率提升。
- 热点内容在多层级延长缓存时长，叠加随机抖动抗雪崩。

**代价**

- 四层缓存一致性维护复杂度上升（失效、回填、层级联动）。
- L1 Caffeine 热门判定逻辑需设计（依赖 hotkey 探测机制）。
- 层级增加，缓存穿透/击穿/雪崩的排查与调试难度上升。