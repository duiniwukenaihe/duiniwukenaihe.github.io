---
layout: post
title: "elasticsearch常用查询"
date: "2019-09-18 10:00:00"
category: kubernetes
tags: elasticsearch
author: duiniwukenaihe
---
* content
{:toc}

 

## 描述背景：

公司内部做了一套elasticsearch集群，开始主要是用来收集前端客户端报错。然后用 elastalert进行微信报警，后来由于mysql数据库写的比较烂无法查询用户信息统计，客户端埋点数据也收集了一份用做数据分析。

先看下索引的mapping吧，开始早先用的是5版本的有多个type，后来升级为6就保留了1个type。7版本貌似type取消了。索引格式为 项目名_渠道_年月份

GET *_wx_201909/_mapping

具体的就不贴了。简单说下mapping：



etype      text   可以理解为一个type吧 主要区分不同类型行为的
rdate      text    算是一个日期标识吧，开始的时候弄的，就用来区分日期了
regTime    text    用户注册的日期
gid        text     项目id
openid     text     就是字面意思用户的openid
pid        text     界面上面点击的按钮的标识
serverid   text      区分正式测试环境的标识
uid        long      用户的userid

![elk1.png](/assets/images/elk/elk1.png)   ![elk2.png](/assets/images/elk/elk2.png) 

哈哈这些的都是HG自己定义的 有些其实不太适合，但是查询常用的已经够了，比较乱，可以尽情喷.


下面开始下简单的查询吧：

查询2019年9.16日新增用户：
  ```bash 
{"size": 0,
  "query": {
     "bool": {
      "must": [
                  { "match": { "rdate": "20190916" }},
                  { "match": { "regTime": "20190916"   }}
]
}
},
		"aggs": {
   "新增用户": {
      "cardinality": {
        "field": "uid"
      }
    }
	}
}
  ``` 
![elk3.png](/assets/images/elk/elk3.png) 
查询2019年9.16日活跃用户
  ```bash 
{"size": 0,
  "query": {
     "bool": {
      "must": [
                  { "match": { "rdate": "20190916" }}
]
}
},
		"aggs": {
   "活跃用户": {
      "cardinality": {
        "field": "uid"
      }
    }
	}
}
  ```
![elk4.png](/assets/images/elk/elk4.png) 

