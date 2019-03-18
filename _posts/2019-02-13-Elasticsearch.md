---
layout: post
title: Elasticsearch
date: 2019-02-13 09:42:31
categories: elastic
share: y
excerpt_separator: <!--more-->
---


<!--more-->

## Elasticsearch

### 查询命令

curl 'http://localhost:9200/?pretty'

curl -H "Content-Type: application/json" -XGET 'http://localhost:9200/_count?pretty' -d '
{
    "query": {
        "match_all": {}
    }
}
'

curl -X PUT "localhost:9200/megacorp/employee/2" -H 'Content-Type: application/json' -d'
{
    "first_name" :  "Jane",
    "last_name" :   "Smith",
    "age" :         32,
    "about" :       "I like to collect rock albums",
    "interests":  [ "music" ]
}
'
curl -X PUT "localhost:9200/megacorp/employee/3" -H 'Content-Type: application/json' -d'
{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         35,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}
'

curl -X GET "localhost:9200/megacorp/employee/1"

curl -X GET "localhost:9200/megacorp/employee/_search"

curl -X GET "localhost:9200/megacorp/employee/_search?q=last_name:Smith"

curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
'

curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
'

curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
'

curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
'

curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
'

curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "aggs": {
      "all_interests": {
      	  "terms": { "field": "interests" }
    	}
  	}
}
'

### 通过源码进行安装的ELK

wget tar包解压后直接 `./bin/xxx`启动即可

### 外网访问ELK

#### 阿里云服务器添加安全组

#### 开发服务器的防火墙 

```
sudo ufw status # 如果是inactive， 则通过下面的命令开启：sudo ufw allow OpenSSH & sudo ufw enable   
sudo ufw allow #{port}
```

#### 外网访问Elasticsearch


1. 修改配置文件 conf/elasticsearch.yml

  `network.host: 0.0.0.0  # 监听全部ip，在实际环境中应设置为一个安全的ip`
  之后会报错文件描述符不能低于65536
2. 文件描述符调整

  ```
  修改 /etc/security/limits.conf为
  * soft nofile 65536
  * hard nofile 65536
  * - memlock unlimited
  ```
3. 重新登录进去后，外网即可访问

#### 外网访问Kibana

1. 修改配置文件 conf/kibana.yml  
   `server.host: "0.0.0.0"`
