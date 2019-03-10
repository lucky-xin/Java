# 创建mapping
PUT music
{
    "mappings": {
        "_doc" : {
            "properties" : {
                "suggest" : {
                    "type" : "completion"
                },
                "title" : {
                    "type": "keyword"
                }
            }
        }
    }
}
# 索引document
PUT music/_doc/1?refresh
{
    "suggest" : {
        "input": [ "Nevermind", "Nirvana" ],
        "weight" : 34
    }
}
# 查询
POST music/_search?pretty
{
    "suggest": {
        "song-suggest" : {
            "prefix" : "nir", 
            "completion" : { 
                "field" : "suggest" 
            }
        }
    }
}