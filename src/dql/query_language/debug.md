# 调试

出于调试的目的，可以将查询参数`debug=true`附加到查询。附加这个参数可以让您在响应的扩展键下检索所有实体的`uid`属性以及`server_latency`和`start_ts`信息。

* `parsing_ns`: 解析查询的延迟（以纳秒为单位）。
* `processing_ns`: 处理查询的延迟（以纳秒为单位）。
* `encoding_ns`: 对`JSON`响应进行编码的延迟（以纳秒为单位）。
* `start_ts`: 事务的逻辑开始时间戳。

以`debug`作为查询参数的查询：

``` bash
curl -H "Content-Type: application/dql" http://localhost:8080/query?debug=true -XPOST -d $'{
  tbl(func: allofterms(name@en, "The Big Lebowski")) {
    name@en
  }
}' | python -m json.tool | less
```

返回`uid`和`server_latency`

``` json 
{
  "data": {
    "tbl": [
      {
        "uid": "0x41434",
        "name@en": "The Big Lebowski"
      },
      {
        "uid": "0x145834",
        "name@en": "The Big Lebowski 2"
      },
      {
        "uid": "0x2c8a40",
        "name@en": "Jeffrey \"The Big\" Lebowski"
      },
      {
        "uid": "0x3454c4",
        "name@en": "The Big Lebowski"
      }
    ],
    "extensions": {
      "server_latency": {
        "parsing_ns": 18559,
        "processing_ns": 802990982,
        "encoding_ns": 1177565
      },
      "txn": {
        "start_ts": 40010
      }
    }
  }
}
```

> 注意`GraphQL+-`已重命名为`Dgraph`查询语言（`DQL`）。虽然`application/dql`是`Content-Type`头的首选值，但我们将继续支持`Content-Type: application/graphql+-`，以避免进行破坏性的版本更新。

