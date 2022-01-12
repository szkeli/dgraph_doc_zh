# 插入块 (Upsert Block)

`upsert`块允许在单个请求中执行查询和变更。`upsert`块包含一个查询块和一个或多个变更块。在查询块中定义的变量可以使用`uid`和`val`函数在变更块中使用。

一般情况下，`Upsert Block`的结构如下：

``` dql
upsert {
  query <query block>
  [fragment <fragment block>]
  mutation <mutation block 1>
  [mutation <mutation block 2>]
  ...
}
```

`upsert`块的执行还返回在执行变更之前对数据库状态执行的查询的响应。为了获得最新的结果，我们应该提交变更并执行另一个查询。

### `uid` 函数

`uid`函数允许从查询块中定义的变量中提取`uid`。根据查询块的执行结果，有两种可能的结果：

* 如果变量是空的，即没有节点匹配查询，`uid`函数返回一个新的`uid`，在`set`操作的情况下，因此被视为类似于一个空节点。另一方面，对于`delete/del`操作，它不返回`UID`，因此该操作成为`no-op`，并被静默地忽略。空节点在所有变更块中获得相同的`UID`。
* 如果变量存储一个或多个`uid`, `uid`函数将返回存储在变量中的所有`uid`。在这种情况下，将对返回的所有`uid`执行操作，一次一个。

### `val` 函数

`val`函数允许从值变量中提取值。值变量存储`uid`到对应值的映射。因此，`val(v)`被存储在`N-Quad`中`UID (Subject)`映射中的值所替换。如果变量`v`没有给定`UID`的值，那么这种变更将被静默地忽略。`val`函数也可以用于聚合变量的结果，在这种情况下，变更中的所有`uid`都将用聚合值更新。

## `uid` 函数的使用例子

考虑拥有下面`Schema`的例子：

``` bash
curl localhost:8080/alter -X POST -d $'
  name: string @index(term) .
  email: string @index(exact, trigram) @upsert .
  age: int @index(int) .' | jq
```

现在，假设我们想要创建一个具有`email`和`name`的`user`。我们还希望确保一个电子邮件在数据库中恰好有一个对应的用户。为了实现这一点，我们首先需要用给定的电子邮件查询数据库中是否存在用户。如果用户存在，则使用其`UID`更新`name`信息。如果用户不存在，我们将创建一个新用户并更新电子邮件和名称信息。

我们能够用下面的`upsert block`块达成上面的需求：

``` bash 
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    q(func: eq(email, "user@company1.io")) {
      v as uid
      name
    }
  }

  mutation {
    set {
      uid(v) <name> "first last" .
      uid(v) <email> "user@company1.io" .
    }
  }
}' | jq
```


返回结果：

``` json
{
  "data": {
    "q": [],
    "code": "Success",
    "message": "Done",
    "uids": {
      "uid(v)": "0x1"
    }
  },
  "extensions": {...}
}
```

`upsert`块的查询部分将用户的`UID`和提供的电子邮件存储在变量`v`中。变更部分从变量`v`中提取`UID`，并将名称和电子邮件信息存储在数据库中。如果用户存在，则更新。如果用户不存在，`uid(v)`将被视为一个空白节点，并创建一个新用户，如上所述。

如果我们再次运行相同的变更，数据将会被覆盖，并且不会创建新的`uid`。请注意，当再次执行变更时，结果中的`uid`映射为空，并且数据映射(键`q`)包含在前一个`upsert`中创建的`uid`：

``` json 
{
  "data": {
    "q": [
      {
        "uid": "0x1",
        "name": "first last"
      }
    ],
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {...}
}
```

我们可以实现相同的结果使用`json`数据集如下：


``` bash
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d '
{
  "query": "{ q(func: eq(email, \"user@company1.io\")) {v as uid, name} }",
  "set": {
    "uid": "uid(v)",
    "name": "first last",
    "email": "user@company1.io"
  }
}' | jq
```

现在，我们想为拥有相同电子邮件user@company1.io的相同用户添加年龄信息。我们可以使用upsert块做如下相同的事情：


``` bash 
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    q(func: eq(email, "user@company1.io")) {
      v as uid
    }
  }

  mutation {
    set {
      uid(v) <age> "28" .
    }
  }
}' | jq
```

返回结果：

``` json 
{
  "data": {
    "q": [
      {
        "uid": "0x1"
      }
    ],
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {...}
}
```

在这里，查询块查询电子邮件为user@company1.io的用户。它将用户的uid存储在变量v中，然后突变块使用uid函数从变量v中提取uid，从而更新用户的年龄。

我们可以实现相同的结果使用json数据集如下：

``` bash
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d $'
{
  "query": "{ q(func: eq(email, \\"user@company1.io\\")) {v as uid} }",
  "set":{
    "uid": "uid(v)",
    "age": "28"
  }
}' | jq
```

如果我们只想在用户存在的时候执行变更，我们可以使用[条件Upsert](./conditional_upsert.md)。

## `val` 函数示例

假设我们想把断言年龄迁移到其他年龄。我们可以使用以下突变来做到这一点：

``` bash
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    v as var(func: has(age)) {
      a as age
    }
  }

  mutation {
    # we copy the values from the old predicate
    set {
      uid(v) <other> val(a) .
    }

    # and we delete the old predicate
    delete {
      uid(v) <age> * .
    }
  }
}' | jq
```

返回结果：

``` json 
{
  "data": {
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {...}
}
```

在这里，变量`a`将存储从所有`uid`到其年龄的映射。然后，变更块将每个`UID`对应的年龄值存储在另一个谓词中，并删除年龄谓词。

我们可以实现相同的结果使用json数据集如下：

``` json
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d $'{
  "query": "{ v as var(func: regexp(email, /.*@company1.io$/)) }",
  "delete": {
    "uid": "uid(v)",
    "age": null
  },
  "set": {
    "uid": "uid(v)",
    "other": "val(a)"
  }
}' | jq
```

## 批量删除示例 (Bulk Delete Exmaple)

假设我们想从数据库中删除company1的所有用户。这可以通过upsert块在一个查询中实现，如下所示：

``` bash
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    v as var(func: regexp(email, /.*@company1.io$/))
  }

  mutation {
    delete {
      uid(v) <name> * .
      uid(v) <email> * .
      uid(v) <age> * .
    }
  }
}' | jq
```

我们可以实现相同的结果使用json数据集如下：

``` json 
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d '{
  "query": "{ v as var(func: regexp(email, /.*@company1.io$/)) }",
  "delete": {
    "uid": "uid(v)",
    "name": null,
    "email": null,
    "age": null
  }
}' | jq
```