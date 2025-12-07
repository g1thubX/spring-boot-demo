# Elasticsearch 全文搜索 - 完全学习指南

## 📚 简介

Elasticsearch 是一个强大的分布式搜索和分析引擎，基于 Lucene。它提供了全文搜索、日志分析、实时数据分析等功能。在海量数据中进行快速搜索，Elasticsearch 比数据库快 100 倍以上。

### 为什么学习这个？
- 🎯 **超快搜索：毫秒级的搜索响应时间**
- 🎯 **全文搜索：分词、同义词、模糊搜索**
- 🎯 **可扩展性：支持 PB 级数据**
- 🎯 **实时分析：日志分析、监控告警**

---

## 🎯 核心概念

### 1. Elasticsearch 的三个核心概念

```
Index (索引)
  ├─ 相当于数据库中的表
  └─ 包含多个 Document

Document (文档)
  ├─ 相当于数据库中的行
  ├─ JSON 格式
  └─ 有唯一的 _id

Mapping (映射)
  ├─ 定义文档结构和字段类型
  └─ 相当于数据库的 schema
```

### 2. 索引操作

```java
@Configuration
public class ElasticsearchConfig {
    
    @Bean
    public RestHighLevelClient restHighLevelClient() {
        return new RestHighLevelClient(
            RestClient.builder(
                new HttpHost("localhost", 9200, "http")
            )
        );
    }
}

@Service
public class ElasticsearchService {
    
    @Autowired
    private RestHighLevelClient client;
    
    // 创建索引
    public void createIndex(String indexName) throws IOException {
        CreateIndexRequest request = new CreateIndexRequest(indexName);
        
        // 设置字段映射
        request.mapping(XContentFactory.jsonBuilder()
            .startObject()
                .startObject("properties")
                    .startObject("title")
                        .field("type", "text")
                    .endObject()
                    .startObject("content")
                        .field("type", "text")
                    .endObject()
                    .startObject("createTime")
                        .field("type", "date")
                    .endObject()
                .endObject()
            .endObject()
        );
        
        client.indices().create(request, RequestOptions.DEFAULT);
    }
    
    // 删除索引
    public void deleteIndex(String indexName) throws IOException {
        DeleteIndexRequest request = new DeleteIndexRequest(indexName);
        client.indices().delete(request, RequestOptions.DEFAULT);
    }
}
```

### 3. 文档操作

```java
@Service
public class DocumentService {
    
    @Autowired
    private RestHighLevelClient client;
    
    // 添加文档
    public void addDocument(Article article) throws IOException {
        IndexRequest request = new IndexRequest("articles");
        request.id(article.getId().toString());
        request.source(JSON.toJSONString(article), XContentType.JSON);
        
        client.index(request, RequestOptions.DEFAULT);
    }
    
    // 删除文档
    public void deleteDocument(String indexName, String id) throws IOException {
        DeleteRequest request = new DeleteRequest(indexName, id);
        client.delete(request, RequestOptions.DEFAULT);
    }
    
    // 更新文档
    public void updateDocument(String id, Map<String, Object> data) throws IOException {
        UpdateRequest request = new UpdateRequest("articles", id);
        request.doc(data);
        
        client.update(request, RequestOptions.DEFAULT);
    }
}
```

### 4. 搜索查询

```java
@Service
public class SearchService {
    
    @Autowired
    private RestHighLevelClient client;
    
    // 全文搜索
    public List<Article> fullTextSearch(String keyword) throws IOException {
        SearchRequest request = new SearchRequest("articles");
        
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.matchQuery("content", keyword));
        request.source(sourceBuilder);
        
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        
        return parseResponse(response);
    }
    
    // 精确搜索
    public List<Article> termSearch(String field, String value) throws IOException {
        SearchRequest request = new SearchRequest("articles");
        
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.termQuery(field, value));
        request.source(sourceBuilder);
        
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        
        return parseResponse(response);
    }
    
    // 范围搜索
    public List<Article> rangeSearch(String field, Object from, Object to) throws IOException {
        SearchRequest request = new SearchRequest("articles");
        
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.rangeQuery(field)
            .gte(from)
            .lte(to));
        request.source(sourceBuilder);
        
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        
        return parseResponse(response);
    }
    
    // 复杂查询（多条件）
    public List<Article> complexSearch(String keyword, Long userId, Date startDate) 
            throws IOException {
        SearchRequest request = new SearchRequest("articles");
        
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        
        // 结合多个查询条件
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        boolQuery.must(QueryBuilders.matchQuery("content", keyword));
        boolQuery.filter(QueryBuilders.termQuery("userId", userId));
        boolQuery.filter(QueryBuilders.rangeQuery("createTime").gte(startDate));
        
        sourceBuilder.query(boolQuery);
        sourceBuilder.from(0);  // 分页
        sourceBuilder.size(20);
        
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        
        return parseResponse(response);
    }
}
```

### 5. 聚合分析

```java
// 分组统计
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
sourceBuilder.aggregation(
    AggregationBuilders.terms("categoryAgg")
        .field("category")
        .size(10)
);

// 日期直方图（按日期分组）
sourceBuilder.aggregation(
    AggregationBuilders.dateHistogram("dateAgg")
        .field("createTime")
        .calendarInterval(DateHistogramInterval.DAY)
);
```

---

## 💡 实现细节

### 1. 分析器（Analyzer）

```java
// 自定义分析器
CreateIndexRequest request = new CreateIndexRequest("products");
request.mapping(XContentFactory.jsonBuilder()
    .startObject()
        .startObject("properties")
            .startObject("name")
                .field("type", "text")
                .field("analyzer", "ik_max_word")  // IK 分词器
                .field("search_analyzer", "ik_smart")
            .endObject()
        .endObject()
    .endObject()
);

// IK 分词器：支持中文分词
// ik_max_word：最细粒度分词
// ik_smart：最粗粒度分词
```

### 2. 字段类型

```
text - 全文本，会被分词
keyword - 关键词，不分词
integer, long, float, double - 数值类型
boolean - 布尔类型
date - 日期类型
geo_point - 地理位置
nested - 嵌套对象
```

---

## 🏢 行业应用

### 1. 电商商品搜索

```java
@Service
public class ProductSearchService {
    
    // 用户搜索商品
    public List<Product> search(String keyword, String category, 
                                BigDecimal minPrice, BigDecimal maxPrice) {
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        
        // 标题和描述中包含关键词
        boolQuery.must(QueryBuilders.multiMatchQuery(keyword, "title", "description"));
        
        // 类别精确匹配
        if (category != null) {
            boolQuery.filter(QueryBuilders.termQuery("category", category));
        }
        
        // 价格范围
        if (minPrice != null || maxPrice != null) {
            RangeQueryBuilder priceRange = QueryBuilders.rangeQuery("price");
            if (minPrice != null) priceRange.gte(minPrice);
            if (maxPrice != null) priceRange.lte(maxPrice);
            boolQuery.filter(priceRange);
        }
        
        // 执行搜索
        return executeSearch(boolQuery);
    }
}
```

### 2. 日志分析

```
日志 → Logstash → Elasticsearch → Kibana
     收集         存储           可视化
```

---

## 🚀 快速开始

### 1. 启动 Elasticsearch

```bash
docker run -d --name elasticsearch \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  docker.elastic.co/elasticsearch/elasticsearch:7.14.0
```

### 2. 依赖

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>7.14.0</version>
</dependency>
```

### 3. 配置和使用

```java
@Configuration
public class ElasticsearchConfig {
    
    @Bean
    public RestHighLevelClient client() {
        return new RestHighLevelClient(
            RestClient.builder(new HttpHost("localhost", 9200, "http"))
        );
    }
}
```

---

## 🧪 测试

```bash
# 创建索引
curl -X PUT http://localhost:9200/articles

# 添加文档
curl -X POST http://localhost:9200/articles/_doc \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","content":"This is a test"}'

# 搜索
curl "http://localhost:9200/articles/_search?q=test"
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| Index | 索引，相当于数据库的表 |
| Document | 文档，相当于行 |
| Mapping | 字段映射，定义结构 |
| Analyzer | 分析器，用于分词 |
| Query | 查询，支持多种查询类型 |
| Aggregation | 聚合，用于分析统计 |

---

**恭喜！** 🎉 你已经掌握了 Elasticsearch 的核心功能！
