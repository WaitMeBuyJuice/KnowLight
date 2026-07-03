# ES 高频面试题整理（基于本项目）

> 本文档基于 `zhiguang_be` 项目中 Elasticsearch 的实际使用整理，所有回答均对应代码位置，便于结合实现深挖。

---

## 一、索引与 Mapping 设计

### Q1. 你们 ES 索引的 Mapping 是怎么设计的？为什么 title/body 用 text，tags/status 用 keyword？

**答：** 见 `SearchIndexInitializer.java:33-53`。

- `title` / `body` / `description` 用 `text` 类型 + IK 分词器，需要全文检索
- `tags` / `status` / `content_type` 用 `keyword`，只做精确过滤/聚合，不分词
- `like_count` / `view_count` 等数值字段用 `integer`/`long`，用于 `function_score` 加权计算
- `publish_time` 用 `date`，用于排序
- `title_suggest` 用 `completion` 类型，专门支持联想建议

设计原则：**需要分词匹配的用 text，需要精确匹配/聚合/排序的用 keyword**。混用会导致 term 查询失效或聚合报错。

### Q2. 为什么 title 索引时用 `ik_max_word`，搜索时用 `ik_smart`？

**答：** 见 `SearchIndexInitializer.java:38`。
```java
.analyzer("ik_max_word").searchAnalyzer("ik_smart")
```

- 索引侧 `ik_max_word`：最细粒度切分，"人工智能"切成"人工智能/人工/智能"，**召回率高**
- 搜索侧 `ik_smart`：智能切分，"人工智能"保留为整体，**精确度高**，避免无效匹配

这是 IK 分词器的经典**倒排索引粒度不对称**配置：索引尽量细（多建倒排项），查询尽量准（少触发歧义匹配），在不增加索引体积太多的前提下兼顾召回与精确。

### Q3. 为什么不直接用动态 Mapping，要手动建？

**答：** 见 `SearchIndexInitializer.java:26-57`，启动时 `ensureIndex` 检查索引是否存在，不存在则创建。

动态 Mapping 的问题：
- 字符串字段默认会被映射成 `text` + `keyword` 双字段，浪费空间且 term 查询要带 `.keyword`
- 数值字段类型推断可能不准（如 ID 被推断成 long 但实际需要 keyword）
- 分词器无法指定 IK，会用默认 standard 分词器，中文按单字切分，检索效果差

手动 Mapping 保证字段类型、分词器、索引策略符合业务需求，且**可重现**——重建索引时行为一致。

---

## 二、查询与相关性

### Q4. 你们的搜索相关性是怎么打的分？业务数据和相关性如何结合？

**答：** 见 `SearchServiceImpl.java:68-89`，用 `function_score` 包裹 `bool` 查询。

```java
.functionScore(fs -> fs
    .query(bool查询)  // multi_match 召回
    .functions(fieldValueFactor(like_count, log1p).weight(2.0))
    .functions(fieldValueFactor(view_count, log1p).weight(1.0))
    .boostMode(Sum))
```

- 内层 `multi_match` 跨 `title^3` 和 `body` 做相关性召回，title 加权 3 倍
- 外层 `function_score` 用 `field_value_factor` 把点赞数、浏览数转成加分
- `modifier=log1p` 对计数取对数，**避免热门内容分数爆炸**（10000 赞和 100 赞的差距不该是 100 倍）
- `boost_mode=sum` 把相关性分数和业务加分相加

这样既保证文本相关性，又让互动数据影响排序，是社区内容搜索的常见打法。

### Q5. 为什么 title 加权用 `^3`，body 不加权？这个值怎么定？

**答：** 见 `SearchServiceImpl.java:71`，`fields("title^3", "body")`。

- title 是内容核心摘要，命中权重应高于正文
- body 是大段文本（最多 4000 字，见 `SearchIndexService.java:111`），命中概率高但相关性弱
- `^3` 是经验值，实际需 A/B 测试调整。过大会让"标题蹭关键词但正文无关"的内容排前面，过小则标题命中优势不明显

延伸：也可以用 `boost` 模式或 `constant_score` 配合，但 `^` 前缀加权最简洁，适合字段数少的场景。

### Q6. status=published 这个过滤为什么放在 `filter` 而不是 `must`？

**答：** 见 `SearchServiceImpl.java:72-73`。

- `filter` 不参与相关性打分，只做"要/不要"判断，**性能更好**且会缓存
- `must` 会参与打分，对 `keyword` 字段无意义（term 查询相关性要么 1 要么 0）
- 软删的帖子（status=deleted）不应被搜出，用 filter 排除

ES 优化铁律：**只过滤不打分的条件一律用 filter**，享受缓存和跳过打分的双重收益。

---

## 三、分页

### Q7. 你们的分页是怎么实现的？为什么不用 `from` + `size`？

**答：** 见 `SearchServiceImpl.java:54-59, 96-99`，用 `search_after` 游标分页。

```java
sorts.add(score desc, publish_time desc, like_count desc, view_count desc, content_id desc);
if (afterValues != null) b.searchAfter(afterValues);
```

- `from + size` 的问题：深分页时 ES 要从每个分片取 `from + size` 条归并，`from=10000` 时单次查询要拉 10w+ 文档，**内存和延迟爆炸**，且默认 `max_result_window=10000` 限制
- `search_after` 用上一页最后一条的 sort 值作游标，**每次只取 size 条**，深分页性能恒定
- 游标 Base64URL 编码后返回前端，前端翻页带上即可

代价：不能随机跳页（只能下一页），排序字段必须包含唯一字段（这里用 `content_id` 兜底）保证游标稳定。

### Q8. `search_after` 为什么排序里要加 `content_id`？

**答：** 见 `SearchServiceImpl.java:59`。

游标分页要求**排序键唯一**，否则前几页 sort 值相同的文档在翻页时可能丢失或重复。`score` / `publish_time` / `like_count` 都可能重复，加 `content_id`（主键唯一）作为最后一道排序，保证每条文档的 sort 值组合绝对唯一，游标定位精准。

### Q9. `scroll` 和 `search_after` 都能深分页，区别是什么？你们为什么选后者？

**答：**

- `scroll`：服务端维护快照上下文，适合**全量导出/重建索引**，但占用服务端内存，且有快照过期问题（数据变更不可见）
- `search_after`：无状态，每次查询独立，适合**用户翻页**，反映实时数据

本项目是用户搜索翻页场景，需要实时性（新发布的帖子应能出现在后续页），且无状态更轻量，所以选 `search_after`。`scroll` 一般只用于后台批量导出。

---

## 四、高亮与联想

### Q10. 高亮是怎么实现的？snippet 怎么拼的？

**答：** 见 `SearchServiceImpl.java:91-94, 250-269`。

```java
.highlight(h -> h
    .fields("title", ...)
    .fields("body", ...))
```

- ES 对命中字段返回 `<em>` 包裹的高亮片段
- `buildSnippet` 把 title 高亮片段和 body 高亮片段拼接，title 在前 body 在后
- 拼接结果作为 `description` 返回前端，没有高亮时回退到原文 description

注意：高亮会增加查询开销（要重新对命中片段切分标注），字段多/正文长时需控制 `number_of_fragments` 和 `fragment_size`，本项目用默认值。

### Q11. 联想建议为什么用 `completion` 类型而不是 `suggest` 或前缀查询？

**答：** 见 `SearchIndexInitializer.java:52` + `SearchServiceImpl.java:169-170`。

- `completion` 类型在内存里建 FST（有限状态机），**前缀匹配极快**，延迟毫秒级，适合搜索框实时联想
- 普通 `match_phrase_prefix` 查询要走倒排索引，延迟高且占用查询资源
- `completion` 写入时需指定 `input` 和可选 `weight`，本项目用 title 作为 input（见 `SearchIndexService.java:119-121`）

代价：`completion` 只支持前缀匹配，不支持纠错、同义词。要纠错得用 `term` suggester，本项目没做。

---

## 五、写入与一致性

### Q12. 帖子发布后多久能搜到？怎么保证"立即可搜"？

**答：** 见 `SearchIndexService.java:124-129`，写入时 `refresh(Refresh.WaitFor)`。

- ES 默认 1 秒 refresh 一次，新写入的文档 1 秒后才可搜
- `Refresh.WaitFor` 让写入请求**等待下次 refresh 完成再返回**，保证返回后即可搜
- 代价：每次写入增加 refresh 等待延迟，写入吞吐下降

权衡：搜索场景用户期望"发布即可搜"，可接受写入慢一点。若是批量导入场景应改用 `false` 或批量 refresh。

### Q13. 你们的 ES 索引数据和 MySQL 怎么同步？链路是什么？

**答：** 链路：`MySQL outbox 表 → Canal 订阅 binlog → Kafka(canal-outbox topic) → 消费者 → ES`。

- 业务写 MySQL 时同事务写 outbox 表（发布一致性）
- `CanalKafkaBridge` 订阅 outbox 表 binlog，转发 INSERT/UPDATE 到 Kafka
- `CanalOutboxConsumerSearch` 消费 Kafka 消息，按 entity=knowpost 过滤，调 `SearchIndexService.upsertKnowPost` 写 ES
- 最终一致性，延迟取决于 Canal 拉取间隔 + Kafka 消费延迟，通常秒级

不用同步双写（业务代码里同时写 MySQL 和 ES）的原因：跨存储无法事务，可能出现 MySQL 成功 ES 失败的不一致；outbox + Canal 把"事件发布"与"业务事务"原子化，是更可靠的方案。

### Q14. 启动时索引是空的怎么办？历史数据怎么进 ES？

**答：** 见 `SearchIndexService.java:52-74`，`@PostConstruct ensureBackfill`。

```java
long cnt = es.count(...).count();
if (cnt > 0) return;  // 已有数据，跳过
// 分页拉 MySQL 公开帖子，逐条 upsertKnowPost
```

- 启动时检查索引文档数，为 0 则触发回灌
- 分页（limit=500）拉 MySQL，逐条写入 ES，避免一次性加载OOM
- 异常容错：回灌失败只打 warn 日志，不阻断启动（索引可由后续增量写入动态创建，但 Mapping 不完整）

局限：只在索引完全空时触发，**增量补漏**（如某条消息消费失败漏写）不会自动补，需要人工触发或定时对账。

### Q15. 帖子删除是硬删还是软删？为什么？

**答：** 见 `SearchIndexService.java:140-155`，软删。

```java
doc.put("status", "deleted");
es.index(req);  // 同 ID 覆盖写入
```

- 不删 ES 文档，只更新 status 字段为 deleted
- 搜索时 filter `status=published` 排除（见 Q6）
- 好处：**可恢复**（误删可改回 published），且避免删文档导致的 segment merge 开销
- 同 ID 覆盖写入保证幂等，重复消费不会产生多条记录

---

## 六、计数冗余与性能

### Q16. 为什么把点赞数、收藏数冗余到 ES？查询时实时查 Redis 不行吗？

**答：** 见 `SearchIndexService.java:114-116`，写入 ES 时从 Redis 拉 like/fav 计数冗余进文档。

- 搜索结果一次返回 N 条，若每条都实时查 Redis 计数，**N 次 Redis 调用**放大延迟
- 冗余进 ES 后，搜索结果一次返回含 like_count，零额外查询
- `function_score` 加权也需要 like_count 字段，必须在 ES 内才能参与打分

代价：计数冗余**非实时**——用户点赞后 ES 里的 like_count 要等下次 upsert 才更新。但搜索排序对计数实时性要求不高，秒级~分钟级延迟可接受。详情页的精确计数仍走 Redis（见 `SearchServiceImpl.java:128-129`）。

### Q17. 计数冗余和 Redis 计数不一致怎么办？

**答：** 当前实现的不一致来源：
- 点赞发生在 upsert 之后，ES like_count 比实际少
- upsert 时拉 Redis 是某一时刻快照，期间增量丢失

缓解策略（项目未全部实现）：
- 定时对账任务批量刷新 ES 计数字段
- 点赞事件触发 ES 局部 update（`doc` 局部更新，比全量 upsert 轻）
- 接受搜索场景的弱一致性，详情页/个人页走 Redis 精确值

---

## 七、客户端与异常

### Q18. 你们用的 ES Java 客户端是哪个？为什么不用 Spring Data Elasticsearch？

**答：** 见 `pom.xml:69-77` + `ElasticsearchConfig.java`，用官方 `elasticsearch-java`（新版 Java Client）+ `elasticsearch-rest-client`（底层传输）。

- `elasticsearch-java` 是 ES 官方主推的新版客户端，类型安全、API 直观、与 ES 版本对齐快
- Spring Data Elasticsearch 的 `ElasticsearchRepository` 抽象层滞后于 ES 新特性，复杂查询（function_score、search_after）要用 `ElasticsearchOperations` 拼，反而更绕
- 新版客户端直接面向 DSL，`SearchServiceImpl` 里的链式 builder 写法清晰，贴近原生 JSON DSL

代价：要手写 Bean 配置（见 `ElasticsearchConfig.java`），没有 Spring Data 的自动仓储。

### Q19. 抓取正文时 charset 嗅探是干嘛的？为什么这么复杂？

**答：** 见 `SearchIndexService.java:160-231`。

正文从外部 URL 拉取（`fetchContentSafe`），来源网页 charset 不规范：
- HTTP header 声明 charset 与页面 meta charset 不一致
- header 声明 ISO-8859-1/ASCII 但实际是 UTF-8 或 GBK
- 完全没声明

处理逻辑：
1. 先嗅探 HTML meta charset
2. 没有则按 header charset
3. header 是可疑值（ISO-8859-1/ASCII）时，对 UTF-8 / GB18030 / header 三种解码统计 `\uFFFD` 替换符数量，取最少的

目的是**避免乱码进 ES 索引**——乱码一旦建索引，搜索召回率和展示都受影响，事后修复要重建索引，成本高。

### Q20. ES 写入失败怎么处理？会丢数据吗？

**答：** 见 `SearchIndexService.java:132-134`，`upsertKnowPost` catch 异常只打 error 日志。

- 当前实现：ES 写入失败**静默吞掉**，不重试不告警，依赖下次帖子变更触发 upsert 补
- Kafka 消费端 `CanalOutboxConsumerSearch` catch 异常也吞掉，`ack.acknowledge()` 提交位点（见 `CanalOutboxConsumerSearch.java:61`），消息不会重投
- 风险：单次 ES 写入失败会导致该帖子索引过期，直到下次元数据变更才补

改进方向：失败时 `ack.acknowledge()` 改为不确认 + 重试，或落本地失败表异步补偿。当前是**优先保证消费吞吐**的取舍。

---

## 八、向量检索（RAG 扩展）

### Q21. 你们 ES 还用了向量检索？和普通搜索什么关系？

**答：** 见 `application.yml:61-65` + `RagIndexService.java`。

- 另建索引 `zhiguang-ai-index`，1536 维向量字段（text-embedding-v4 模型）
- 用于 RAG（检索增强生成）场景，语义检索 + LLM 问答
- 与内容搜索索引 `zhiguang_content_index` 分离，各管各的

向量检索走 `knn` 查询，按语义相似度召回，适合"意思相近但关键词不同"的场景；关键词搜索走 BM25，适合精确匹配。两者互补，本项目分开索引避免互相干扰。

---

## 九、高频追问汇总

### Q22. 如果让你优化这套 ES 方案，你会改哪里？

可从以下角度答：
1. **写入容错**：ES 写入失败目前静默吞，应加重试或失败补偿表
2. **计数实时性**：点赞事件触发 ES 局部 update，而非等下次全量 upsert
3. **去重**：消费端用 outbox 的 outId 做幂等键（当前 search consumer 裸奔，靠 ES upsert 幂等兜底）
4. **索引重建**：Mapping 变更需要重建索引，当前无别名 + reindex 方案，停机风险
5. **监控**：缺少 ES 查询延迟、写入失败率、索引体积的监控指标
6. **分片规划**：单索引单分片，数据量大时需规划分片数和副本

### Q23. ES 集群挂了搜索怎么办？有降级方案吗？

**答：** 当前 `SearchServiceImpl.java:103-105` catch 异常返回空列表，是**静默降级**。

更完整的降级：
- 热门查询结果缓存（Redis/Caffeine），ES 挂时返回缓存
- 降级到 MySQL like 查询（召回差但可用）
- 前端提示"搜索服务降级"

本项目未实现，面试可作改进点提。

---

## 附：项目 ES 关键文件索引

| 文件 | 职责 |
|---|---|
| `SearchIndexInitializer.java` | 索引与 Mapping 初始化 |
| `SearchIndexService.java` | 文档 upsert/软删/回灌/正文抓取 |
| `SearchServiceImpl.java` | 搜索/联想查询逻辑 |
| `SearchController.java` | 搜索 API 入口 |
| `CanalOutboxConsumerSearch.java` | Kafka→ES 增量同步消费者 |
| `ElasticsearchConfig.java` | ES 客户端 Bean 配置 |
| `RagIndexService.java` | RAG 向量索引服务 |
