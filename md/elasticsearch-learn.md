# 查询结果只返回englishName，chineseName字段
```text
GET smart_sourcing_system_company_search_index/_search
{
  "query": {
    "match": {
      "englishName": "Sogou, Inc."
    }
  },
  "_source": ["englishName", "chineseName"]
}

GET /_search
{
    "_source": [ "obj1.*", "obj2.*" ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}

```