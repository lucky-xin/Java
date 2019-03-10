# reindex可以将同一个字段改为不同类型
PUT my_index/_doc/1
{
  "date" : "2018-10-26 00:00:00",
  "title":"中国"
}

PUT new_my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "date": {
          "type": "date",
          "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||yyyy-MM-dd'T'HH:mm:ss"
        },
        "title":{
          "type": "keyword"
        }
      }
    }
  }
}

POST _reindex
{
  "source": {
    "index": "my_index"
  },
  "dest": {
    "index": "new_my_index"
  }
}
同步远程的index到本机
POST _reindex
{
  "source": {
     "remote": {
      "host": "http://127.0.0.1:9200",
      "socket_timeout": "30s",
      "connect_timeout": "30s"
    },
    "index": "my_index",
    "size": 2000,
    "query": {}
  },
  "dest": {
    "index": "my_index"
  }
}
