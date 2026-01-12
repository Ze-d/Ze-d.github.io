---
title: ES简介
date: 2026-01-04 17:18:12
tags: elastic search
---

## 1. 核心架构与原理 (Architecture)

Elasticsearch (ES) 是基于 **Apache Lucene** 构建的分布式搜索和分析引擎。

### 1.1 物理架构

- **Cluster (集群):** 由一个或多个节点组成，共享同一个 Cluster Name。
- **Node (节点):** 单个 ES 实例。
  - **Master Node:** 负责集群元数据管理（创建索引、节点监控）。
  - **Data Node:** 负责数据存储、CRUD、聚合计算（IO 和 CPU 密集型）。
  - **Coordinating Node:** 负责路由请求、合并结果（内存密集型）。
- **Shard (分片):** 索引数据的水平拆分。
  - **Primary Shard:** 负责写操作。数量在索引创建时指定，**不可更改**。
  - **Replica Shard:** 主分片的副本，提供读扩容和高可用。数量可动态调整。

### 1.2 数据存储原理

- **Inverted Index (倒排索引):** 用于全文检索 (`text` 类型)。
  - 结构：`Term (词项) -> Document ID List (文档ID列表)`。
- **Doc Values (列式存储):** 用于排序、聚合 (`keyword` / `numeric` / `date`)。
  - 结构：`Document ID -> Field Value`，存储在磁盘，利用 OS Page Cache。
- **Segment (段):** 分片的最小物理存储单元。写入的数据先进入内存 Buffer，定期刷写到磁盘形成不可变的 Segment。

------

## 2. 环境部署与配置 (Deployment)

推荐使用 Docker/Kubernetes 进行容器化部署。

### 2.1 Docker Compose 快速启动 (单机开发版)

YAML

```
version: '3'
services:
  elasticsearch:
    image: elasticsearch:8.11.0
    container_name: es-node01
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=false # 开发环境关闭安全验证
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://es-node01:9200
    depends_on:
      - elasticsearch

volumes:
  es_data:
```

### 2.2 生产环境关键配置 (`elasticsearch.yml`)

- `cluster.name`: 生产环境必须唯一。
- `node.name`: 节点标识。
- `path.data`: 挂载高性能磁盘（SSD）。
- `bootstrap.memory_lock`: 设置为 `true`，禁止 SWAP 交换内存。
- `discovery.seed_hosts`: 集群节点列表。

------

## 3. 数据建模 (Data Modeling)

**Mapping (映射)** 相当于数据库的 Schema。**严禁在生产环境依赖 Dynamic Mapping (自动推断)，必须显式定义。**

### 3.1 常用数据类型

| **类型**          | **说明** | **适用场景**                                 |
| ----------------- | -------- | -------------------------------------------- |
| **text**          | 会分词   | 全文检索（文章内容、描述）                   |
| **keyword**       | 不分词   | 精确匹配、过滤、排序、聚合（状态、ID、标签） |
| **long / double** | 数值     | 价格、库存、数量                             |
| **date**          | 日期     | 严格遵循 ISO8601 格式                        |
| **boolean**       | 布尔     | `true` / `false`                             |
| **geo_point**     | 地理坐标 | 经纬度查询                                   |
| **nested**        | 嵌套对象 | 数组中包含独立对象时使用                     |

### 3.2 映射定义示例 (含中文分词)

*注意：需预先安装 `analysis-ik` 插件。*

JSON

```
PUT /orders_v1
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "order_id": { "type": "keyword" },
      "product_name": { 
        "type": "text", 
        "analyzer": "ik_max_word",    // 写入时最大化分词
        "search_analyzer": "ik_smart" // 查询时智能分词
      },
      "price": { "type": "double" },
      "created_at": { "type": "date" },
      "tags": { "type": "keyword" }
    }
  }
}
```

------

## 4. 查询语法规范 (Query DSL)

ES 查询分为 **Leaf Query** (单字段) 和 **Compound Query** (组合)。

### 4.1 核心查询模板 (Bool Query)

这是最通用的查询结构，包含筛选和评分。

JSON

```
GET /orders_v1/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "product_name": "华为手机" }} // 贡献得分 (Score)
      ],
      "filter": [
        { "term": { "status": "paid" }},          // 不算分，有缓存 (高性能)
        { "range": { "price": { "gte": 2000 }}}
      ],
      "should": [
        { "term": { "tags": "vip_user" }}         // 命中则加分，不命中也保留
      ]
    }
  },
  "from": 0,
  "size": 20,
  "sort": [
    { "created_at": { "order": "desc" }}
  ]
}
```

### 4.2 聚合分析 (Aggregations)

用于统计分析（类似于 SQL 的 GROUP BY）。

JSON

```
GET /orders_v1/_search
{
  "size": 0, // 不需要返回具体文档，只看统计
  "aggs": {
    "group_by_status": {
      "terms": { "field": "status" }, // 按状态分组
      "aggs": {
        "avg_price": {
          "avg": { "field": "price" } // 计算每组的平均价格
        }
      }
    }
  }
}
```

------

## 5. 性能调优指南 (Performance Tuning)

### 5.1 写入性能优化 (Write)

1. **Bulk API:** 批量写入，不要单条插入。建议每批 5MB-15MB。
2. **Refresh Interval:** 默认 1s。大量导入数据时，可临时改为 `-1` 或 `30s`，导入完成后恢复。
3. **ID 生成:** 尽量使用自动生成的 ID，避免 ES 检查 ID 是否存在的开销。

### 5.2 查询性能优化 (Read)

1. **Filter Context:** 尽可能使用 `filter` 替代 `must`，除非真的需要按相关度排序。
2. **避免 Deep Paging:** 翻页深度超过 10000 条时，禁止使用 `from/size`。
   - **方案:** 使用 `search_after` API。
3. **Keyword vs Text:** 不需要全文检索的字段一律用 `keyword`。
4. **Force Merge:** 历史不再修改的索引（如上个月的日志），执行 Force Merge 减少 Segment 数量，提升查询速度。

------

## 6. 生产运维规范 (O&M)

### 6.1 常用 CAT 命令

用于快速查看集群状态（类 Linux 命令风格）。

- `GET /_cat/health?v` : 查看集群健康度 (Green/Yellow/Red)。
- `GET /_cat/nodes?v` : 查看节点资源使用情况。
- `GET /_cat/indices?v&s=store.size:desc` : 查看索引列表并按大小排序。

### 6.2 生命周期管理 (ILM)

对于日志类数据，必须配置 ILM (Index Lifecycle Management) 策略：

1. **Hot Phase:** 写入中，使用高性能 SSD。
2. **Warm Phase:** 7天后，只读，缩减副本，迁移到 HDD。
3. **Delete Phase:** 30天后，自动删除索引。

### 6.3 备份与恢复

使用 **Snapshot** 机制，将数据备份到 HDFS/S3/本地文件系统。

JSON

```
PUT /_snapshot/my_backup/snapshot_1
{
  "indices": "orders_v1",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

------

## 7. 客户端接入 (Clients)

官方推荐客户端：

- **Java:** `Elasticsearch Java API Client` (替换旧版的 HighLevelRestClient)。
- **Python:** `elasticsearch-py`
- **Go:** `go-elasticsearch`

**Python 示例:**

Python

```
from elasticsearch import Elasticsearch

es = Elasticsearch("http://localhost:9200")

# 搜索
resp = es.search(index="orders_v1", query={"match": {"product_name": "phone"}})
print(resp['hits']['hits'])
```

