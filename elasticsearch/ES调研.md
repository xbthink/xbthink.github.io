# Elasticsearch 调研
## 问题
makepolo有一张creative表和creative_report表。creative_report表现在一天有30万左右，creative表有几百万。

现在有个需求是查询一年内的创意维度的汇总数据，并且可以根据创意名字和状态进行过滤和支持根据聚合结果排序分页。

目前是根据creative id join两张表来实现，随着数据量越来越多，已经不可能做到秒级查询。

## 解决方案
1. 对数据库进行分库分表
2. 更换存储引擎，调研es的聚合排序分页能力。

### 分库分表

### es调研

#### es索引创建
对于索引创建有两种方式：
1. 所有字段展开冗余存储，每天每个creative id一条doc。也是通常消除join用到的一种方式。
2. 使用[es join字段类型](https://www.elastic.co/guide/en/elasticsearch/reference/7.5/parent-join.html)建索引
   join是一种特殊的字段类型，可以对一个索引内的文档建立父子关系。

   join字段定义和父子doc如下所示，查询参见[children aggretation](https://www.elastic.co/guide/en/elasticsearch/reference/7.5/search-aggregations-bucket-children-aggregation.html)和[Parent Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.5/search-aggregations-bucket-parent-aggregation.html),更详细的说明请看es文档。
```json
PUT my_index
{
  "mappings": {
    "properties": {
      "my_join_field": { 
        "type": "join",
        "relations": {
          "creative": "creative_report" 
        }
      }
    }
  }
}

PUT my_index/_doc/1?refresh
{
  "text": "父doc",
  "my_join_field": {
    "name": "creative" 
  }
}

PUT my_index/_doc/3?routing=1&refresh 
{
  "text": "子doc",
  "my_join_field": {
    "name": "creative_report", 
    "parent": "1" 
  }
}
```
#### es索引查询

es查询主要分为两个部分query和aggregation。

- query相当于sql里的where部分，根据是否参加relevance score计算，分为query context和filter context。
  query可以看做是一个抽象语法树，有叶子查询和组合查询，可以嵌套。

- [aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.5/search-aggregations.html)基于query提供数据聚合。主要分为4种类型：bucketing aggr，metric aggr，pipeline aggr， matrix aggr。

  1. bucketing aggr
     根据key或者分桶规则，把文档分到不同的桶里。
  2. metric aggr
     对一个字段进行聚合计算，如sum，avg，max等
  3. pipeline aggr
     在其他aggr的结果上，做一些聚合。
  4. matrix aggr
     对多个字段进行聚合产生一个矩阵。

针对我们聚合后排序分页，先介绍第一种方式查询如下
```json
{
  "aggs":{
    "creative_buckets": {
      "terms": {
        "field": "id",
        "size": 10
      },
      "aggs":{
        "to-report":{
          "children": {
            "type" : "creative_report" 
          },
          "aggs":{
            "date_filter":{
              "filter": {
                "range": {
                   "date": {
                      "from": "2020-07-01"
                   }
                }
              },
              "aggs":{
                "imp_sum":{
                  "sum": {
                    "field": "impression" 
                  }
                }
              }
            }
          }
        },
        "imp_sum_sort": {
          "bucket_sort": {
            "sort": [
              { "to-report>date_filter>imp_sum": { "order": "desc" } } 
            ],
            "from":0,
            "size": 20
          }
        }
      }
    }
  }
}
```
现根据id进行terms aggr分桶，然后再根据bucket_sort aggr进行分桶后排序分页。（上面的例子还有children aggr，filter aggr，sum aggr）

terms aggr根据一个字段，每个值一个分桶。
size 10算法为每个分桶取top 10，协调节点在进行合并返回最终的top10.

这种算法有两个问题
- size不可能指定太大，并且terms不能分页。
- terms返回是不精确的。
  
bucket_sort aggr是一个pipeline aggr，可以对多桶的父aggr进行排序分页。

根据上面的解释，我们可以对查询执行聚合排序分页，但是要求分桶数据量较小。




我们再看一下es sql是如何做聚合排序分页的。sql如下
```sql
SELECT creative_id,sum(impression) as imp FROM creative_report_test3 where date>='2020-08-01' group by creative_id ORDER BY imp DESC limit 100
```

对应的es查询
```json
curl -X POST "192.168.28.99:9200/_sql/translate?pretty" -H 'Content-Type: application/json' -d'
{
  "query": "SELECT creative_id,sum(impression) as imp FROM creative_report_test3 where date>='2020-08-01' group by creative_id ORDER BY imp DESC limit 100"
}
'

{
  "size" : 0,
  "query" : {
    "range" : {
      "date" : {
        "from" : 2011,
        "to" : null,
        "include_lower" : true,
        "include_upper" : false,
        "boost" : 1.0
      }
    }
  },
  "_source" : false,
  "stored_fields" : "_none_",
  "aggregations" : {
    "groupby" : {
      "composite" : {
        "size" : 1000,
        "sources" : [
          {
            "494" : {
              "terms" : {
                "field" : "creative_id",
                "missing_bucket" : true,
                "order" : "asc"
              }
            }
          }
        ]
      },
      "aggregations" : {
        "505" : {
          "sum" : {
            "field" : "impression"
          }
        }
      }
    }
  }
}
```
简单介绍下es查询过程，首先进行range过滤date>='2020-08-01'。
然后基于过滤结果，根据creative_id进行[composite](https://www.elastic.co/guide/en/elasticsearch/reference/7.5/search-aggregations-bucket-composite-aggregation.html)分桶，再在每个桶上做sum(impression).
最后根据sum(impression)对桶进行排序（sql排序是在协调节点上对最后的结果集上进行内存排序，所以es查询里没有对应的模块）。

下面主要介绍一下composite分桶的特点
- 可以根据多个字段进行分桶
- 可以根据分桶的字段进行排序
- 可以对所有的桶进行分页，类似scroll。提供了after，size配置项
- 对于数组字段，可以对数组的值组合，进行分桶。如[1,2] [a,b],分桶为[1,a] [1,b] [2,a] [2,b]4个桶
- composite目前不兼容pipeline aggr。
