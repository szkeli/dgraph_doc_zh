# 原生 `HTTP` （Raw HTTP）

> 原生`HTTP`接口比我们的客户端接口的使用更复杂。我们编写这篇指南是为了帮助您用一种新的语言构建`Dgraph`客户机。

你可以通过`HTTP`接口直接和`Dgraph`交互。这允许你为无法实现`gRPC`的平台构建`Dgraph`客户端。

下面的这些例子中，使用了常用的命令行工具，比如`curl`和`jq`。使用它们的目的是向实现`Dgraph`客户端展示如何通过`HTTP`接口与`Dgraph`交互。

如果你想了解`gRPC`客户端的细节，请参考`GO`的`Dgraph`客户端的实现。

与`GO`客户端的例子相同，现在假设一个银行转账的例子：

## 创建一个客户端

构建在`HTTP` `API`之上的客户端需要为每个事务跟踪三个状态。

1. 开始时间戳`start_ts`。这是一个独一无二的事务`ID`，并且在整个事务的生命周期中不会改变。
2. 由事务（`keys`）修改的键的集合。这有助于事务冲突检测。每个变更都会发回一组新的`keys`。客户端必须将它们与现有的集合合并。或者，客户端可以在合并时取消这些`keys`。
3. 事务中修改的`predicate`的集合`preds`，这有助于谓词移动检测。每个变更都会传回一组新的`preds`。客户端必须将它们与现有的集合合并。或者，客户端可以在合并时取消这些`keys`。

## 修改数据库

`/alter`接口用户创建或者修改数据库的`schema`，如下，`name`是账户的名字，为它设立有一个`term`索引，可以方便我们搜索它：

``` bash
curl -X POST localhost:8080/alter -d \
'name: string @index(term) .
type Person {
   name
}'
```

如果正常的话，上面的脚本应该返回：```{"code": "Success", "message": "Done"}```

删除一个`predicate`或者整个数据库的操作也是向`/alter`接口提交的。

比如，删除`name`这个`predicate`：

``` bash
curl -X POST localhost:8080/alter -d '{"drop_attr": "name"}'
```

再比如，删除`File`这个`type`：

``` bash
curl -X POST localhost:8080/alter -d '{"drop_op": "TYPE", "drop_value": "Film"}'
```

删除所有数据和`schema`：

``` bash
curl -X POST localhost:8080/alter -d '{"drop_all": true}'
```

只删除数据，不删除`schema`：

``` bash
curl -X POST localhost:8080/alter -d '{"drop_op": "DATA"}'
```

## 使用事务

假设我们已经有了一些具有金额的初始银行账户，我们希望从一个账户转账到另一个账户。需要下面四个步骤：

1. 创建一个新的事务
2. 在事务里面，使用`query`查询到当前的账户余额。
3. 使用一个`mutation`来更新余额
4. 提交这个事务。

开启一个事务并不会和`Dgraph`进行任何的交互。事务需要初始化一些状态：`start_us`可以设置为`0`，`keys`可以被设置为一个空集。

**对于`query`或者`mutation`，如果参数中提供了`start_us`，那么该操作将作为正在进行的事务的一部分执行。否则，将启动一个新事务。**

## 使用 `query`

你可以通过`/query`接口查询数据库。注意设置`Content-Type`为`application/dql`来使`dql`能被正确解析。

> 注意：`GraphQL+-`已经被更名为`DQL`，但是我们仍然支持`Content-Type: application/graphql+-`

获取账户余额：

``` bash
curl -H "Content-Type: application/dql" -X POST localhost:8080/query -d $'
{
  balances(func: anyofterms(name, "Alice Bob")) {
    uid
    name
    balance
  }
}' | jq
```

上面的脚本将返回下面这样的结果：

``` json
{
  "data": {
    "balances": [
      {
        "uid": "0x1",
        "name": "Alice",
        "balance": "100"
      },
      {
        "uid": "0x2",
        "name": "Bob",
        "balance": "70"
      }
    ]
  },
  "extensions": {
    "server_latency": {
      "parsing_ns": 70494,
      "processing_ns": 697140,
      "encoding_ns": 1560151
    },
    "txn": {
      "start_ts": 4,
    }
  }
}
```

注意，与数据字段下的查询结果一起的是`extensions -> txn`的其他数据。客户端必须跟踪这些数据。

对于查询，结果中存在`start_us`，这个`start_ts`将需要在此事务与`Dgraph`的所有后续交互中使用，因此是事务状态的一部分。


## 使用变更 `mutation`

现在，我们拥有了用户的余额，那下一步就需要更新用户的余额。下面的`RDF`展示了`Bob`转账`10$`给`Alice`：

``` rdf
<0x1> <balance> "110" .
<0x1> <dgraph.type> "Balance" .
<0x2> <balance> "60" .
<0x2> <dgraph.type> "Balance" .
```

注意，我们必须以`RDF`格式的`UID`引用`Alice`和`Bob`节点。

`mutation`可以提交到`/mutation`接口。这个提交必须携带一个`start_us`作为一个参数，这样`Dgraph`就知道这个变更是哪一个事务的。注意，需要指定`Content-Type`为`application/rdf`以表明你提交的数据是`rdf`格式的：

``` bash
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?startTs=4 -d $'
{
  set {
    <0x1> <balance> "110" .
    <0x1> <dgraph.type> "Balance" .
    <0x2> <balance> "60" .
    <0x2> <dgraph.type> "Balance" .
  }
}
' | jq
```

返回如下结果：


``` json 
{
  "data": {
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {
    "server_latency": {
      "parsing_ns": 50901,
      "processing_ns": 14631082
    },
    "txn": {
      "start_ts": 4,
      "keys": [
        "2ahy9oh4s9csc",
        "3ekeez23q5149"
      ],
      "preds": [
        "1-balance"
      ]
    }
  }
}
```

我们得到了一些`keys`。这些应该添加到存储在事务状态中的`keys`中。我们还获得了一些`preds`，应该将它们添加到存储在事务状态中的`preds`集合中。


## 提交此事务


> 注意，可以在发生变化后立即提交(不需要使用`/commit`端点，如本节所述)。为此，在`URL /mutate?commitNow=true`中添加参数`commitNow`。

