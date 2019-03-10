# nested优势
PUT object_index/_doc/1
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}

会被存储为
{
  "group" :        "fans",
  "user.first" : [ "alice", "john" ],
  "user.last" :  [ "smith", "white" ]
}

GET object_index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "user.first": "Alice" }},
        { "match": { "user.last":  "Smith" }}
      ]
    }
  }
}
修改为nested
PUT nested_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "user": {
          "type": "nested" 
        }
      }
    }
  }
}

PUT nested_index/_doc/1
{
  "group" : "fans",
  "user" : [
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
再进行查询
此时查询必须为nested查询，path为主文档key。普通查询nested类型文档会被过滤掉。不会被搜索到，包括QueryString
GET nested_index/_search
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "Smith" }} 
          ]
        }
      }
    }
  }
}

GET nested_index/_search
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "White" }} 
          ]
        }
      },
      "inner_hits": { 
        "highlight": {
          "fields": {
            "user.first": {}
          }
        }
      }
    }
  }
}

此时QueryString搜索不到值
GET nested_index/_search
{
    "query": {
        "query_string" : {
            "default_field" : "user.first",
            "query" : "Alice"
        }
    }
}

一下指定搜索为nested搜索才能搜索到值
GET nested_index/_search
{
	"query": {
		"nested": {
			"path": "user",
			"query": {
				"query_string": {
					"default_field": "user.first",
					"query": "Alice"
				}
			}
		}
	}
}

# 对于nested类型，嵌套的子json是单独存储的。所以查询出来的是一整个嵌套结构。nested还有一个好处是inner hit。
# 如果我们使用Nested Query时
```text
mapping为
"pdfResource": {
  "type": "nested",
  "properties": {
    "filePath": {
      "type": "text",
      "analyzer": "ik_max_word",
      "search_analyzer": "ik_smart"
    },
    "id": {
      "type": "text",
      "fields": {
        "keyword": {
          "type": "keyword",
          "ignore_above": 256
        }
      }
    },
    "shapes": {
      "properties": {
        "objects": {
          "properties": {
            "fontSize": {
              "type": "long"
            },
            "value": {
              "type": "text",
              "analyzer": "ik_max_word",
              "search_analyzer": "ik_smart"
            }
          }
        }
      }
    }
  }
}
查询为
GET smart_sourcing_system_company_search_index/_search
{
  "query": {
    "nested": {
      "path": "pdfResource",
      "query": {
        "match": {
          "pdfResource.shapes.objects.value": "TOYODA"
        }
      },
      "inner_hits" : {}
    }
  }
}
查询结果
{
	"took": 96,
	"timed_out": false,
	"_shards": {
		"total": 1,
		"successful": 1,
		"skipped": 0,
		"failed": 0
	},
	"hits": {
		"total": 2,
		"max_score": 7.710449,
		"hits": [{
			"_index": "smart_sourcing_system_company_search_index",
			"_type": "_doc",
			"_id": "22",
			"_score": 7.710449,
			"_source": {
				"englishName": "Toyota Motor Corporation",
				"companyName": "Toyota Motor Corporation",
				"editTime": "2019-01-10 16:00:39",
				"type": "Public company",
				"uuid": "05f0dbd3027cad374d5363a0a8d39d9e",
				"id": 22,
				"location_cn": "全世界",
				"englishSimName": "Toyota",
				"foundingYear": "1937-08-28",
				"pdfResource": [],
				"inner_hits": {
					"pdfResource": {
						"hits": {
							"total": 5,
							"max_score": 8.411463,
							"hits": [

								{
									"_index": "smart_sourcing_system_company_search_index",
									"_type": "_doc",
									"_id": "22",
									"_nested": {
										"field": "pdfResource",
										"offset": 5
									},
									"_score": 8.380739,
									"_source": {
										"shapes": [{
											"objects": [{
													"fontSize": 118,
													"value": "BMW Visit Agenda"
												},
												{
													"fontSize": 53,
													"value": "TOYODA GOSEI"
												},
												{
													"fontSize": 157,
													"value": "Very Warm We/come BMw"
												},
												{
													"fontSize": 118,
													"value": "TGZS: Narukawa( President)\nBMW"
												},
												{
													"fontSize": 46,
													"value": "Welcome speech"
												},
												{
													"fontSize": 45,
													"value": "1445~14:55"
												},
												{
													"fontSize": 57,
													"value": "Meeting Room"
												},
												{
													"fontSize": 61,
													"value": "14:55~15:10"
												},
												{
													"fontSize": 60,
													"value": "TGZS Company Presentation"
												},
												{
													"fontSize": 57,
													"value": "TGSH Dong Dandan"
												},
												{
													"fontSize": 58,
													"value": "15:10~15:15"
												},
												{
													"fontSize": 46,
													"value": "Change safety shoes clothes"
												},
												{
													"fontSize": 60,
													"value": "TGZS Plant tour"
												},
												{
													"fontSize": 61,
													"value": "15:15~16:45"
												},
												{
													"fontSize": 52,
													"value": "Plant"
												},
												{
													"fontSize": 46,
													"value": "TGZS: Qian Jie"
												},
												{
													"fontSize": 54,
													"value": "16:45~17:15"
												},
												{
													"fontSize": 58,
													"value": "Summary and close meeting"
												},
												{
													"fontSize": 56,
													"value": "Meeting Room"
												},
												{
													"fontSize": 45,
													"value": "BMW/TGZS"
												},
												{
													"fontSize": 68,
													"value": "17:15~20:00"
												},
												{
													"fontSize": 61,
													"value": "TGzS→ Shanghai"
												},
												{
													"fontSize": 50,
													"value": "GSH: Hara/Dong dandan"
												},
												{
													"fontSize": 30,
													"value": "ove"
												}
											]
										}],
										"filePath": "/home/czm/data/fileData/20170207_【TGZS】TOYODA GOSEI Overview For BMW-(20170207).pdf",
										"id": ""
									}
								}
							]
						}
					}
				}
			}
		}]
	}
}
查询结果会把匹配的嵌套结构存入inner_hits之中返回这样可以根据inner_hits之中信息拿到filePath从而拿到pdf文件名称。
这样就可以把该pdf名称返回到前端。就可以告诉用户是哪个PDF匹配了。
```

