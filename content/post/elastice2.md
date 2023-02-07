---
title: "ElasticSearch（基础操作）"
date: 2023-02-07T15:52:40+08:00
draft: false
description: "基础操作"
tags: ["ElasticSearch"]
categories: ["ElasticSearch"]
---

* 设置索引分片

```
PUT /blogs
{
   "settings" : {
	//设置3个主分片
      "number_of_shards" : 3,
	//设置1个副分片
      "number_of_replicas" : 1
   }
}
```

一个分片保存所有数据的一部分
副分片是主分片的一个拷贝备份，同时用于搜索和返回文档
主分片在索引创建时指定，不能被修改，副分片可以被修改

* 使用自定义ID索引文档

```
PUT /{index}/{type}/{id}
{
  "field": "value",
  ...
}
```

* 使用ElasticSearch生成ID索引文档

```
POST /{index}/{_type}/
{
  "title": "My second blog entry",
  "text":  "Still trying this out...",
  "date":  "2014/01/01"
}
```

将请求修改为POST,URL不指定ID，Es会为文档自动生成ID

* 获取一个文档

```
GET /{index}/{_type}/{id}
```

* 获取文档的部分字段

```
GET /{index}/{_type}/{id}?_source={filed}，{filed}
```

* 获取文档source

```
GET /{index}/{_type}/{id}/_source
```

* 检测文档是否存在

```
XHEAD /{index}/{_type}/{id}
```
文档如果存在，Es会返回`200 ok`的响应码
如果不存在，会返回`404`

* 更新整个文档
```
PUT  /{index}/{_type}/{id}
{
  "title": "My first blog entry",
  "text":  "I am starting to get the hang of this...",
  "date":  "2014/01/02"
}
```
Es中所有文档都是不可修改的，当我们进行更新的时候，Es内部会将文档进行删除，将新的文档覆盖到旧文档的位置，并且更新`_version`版本号
流程为
1.从旧文档构建 JSON
2.更改该 JSON
3.删除旧文档
4.索引一个新文档

* 使用指定ID创建一个新的文档两种方式
```
PUT /{index}/{_type}/{id}?op_type=create
{ ... }
```
```
PUT /{index}/{_type}/{id}/_create
{ ... }
```
两个方式效果一样，如果文档不存在，Es成功索引了文档，会返回元数据和一个 `201 Created` 的 HTTP 响应码。
如果具有相同的 `_index` 、 `_type` 和 `_id` 的文档已经存在，Elasticsearch 将会返回 `409 Conflict` 响应码

* 删除文档
```
DELETE /{index}/{_type}/{id}
```
如果文档存在，Es会返回一个 `200 ` 的 HTTP 响应码。
不存在，Es会返回一个 `404 ` 的 HTTP 响应码。

删除文档不会立即将文档从磁盘中删除，只是将文档标记为已删除状态。随着你不断的索引更多的数据，Elasticsearch 将会在后台清理标记为已删除的文档。

* 轻量搜索
```
GET /{index}/{_type}/_search?q=filed:value
```
如果文档存在，Es会返回一个 `200 ` 的 HTTP 响应码。
不存在，Es会返回一个 `404 ` 的 HTTP 响应码。
