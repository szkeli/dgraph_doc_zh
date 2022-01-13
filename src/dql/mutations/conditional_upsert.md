# 条件插入

插入块(upsert block)支持使用`@if`指令指定一个条件变更块(mutation block)。这个变更块只有在指定的条件为`true`时才会执行。如果指定的条件是`false`，`Dgraph`会静默忽略该变更块(mutation block)，通常一个条件变更块有下面这样的结构：

``` dql
upsert {
  query <query block>
  [fragment <fragment block>]
  mutation [@if(<condition>)] <mutation block 1>
  [mutation [@if(<condition>)] <mutation block 2>]
  ...
}
```

`@if`指令接受一个定义在查询块(query block)的条件变量，里面的条件变量能够和`AND` `OR` `NOT` 一起使用。

## 一个使用条件插入的小例子

假设在我们之前的例子中，我们知道公司1的员工少于100人。为了安全起见，我们希望只在变量v中存储的uid小于100但大于50时才执行变更。这可以实现如下：

``` bash
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d  $'
upsert {
  query {
    v as var(func: regexp(email, /.*@company1.io$/))
  }

  mutation @if(lt(len(v), 100) AND gt(len(v), 50)) {
    delete {
      uid(v) <name> * .
      uid(v) <email> * .
      uid(v) <age> * .
    }
  }
}' | jq
```

我们可以实现相同的结果使用json数据集如下：

``` bash
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d '{
  "query": "{ v as var(func: regexp(email, /.*@company1.io$/)) }",
  "cond": "@if(lt(len(v), 100) AND gt(len(v), 50))",
  "delete": {
    "uid": "uid(v)",
    "name": null,
    "email": null,
    "age": null
  }
}' | jq
```

## 关于多变更块的小例子

考虑以下`Schema`的一个例子：

``` bash
curl localhost:8080/alter -X POST -d $'
  name: string @index(term) .
  email: [string] @index(exact) @upsert .' | jq
```

假设，我们的数据库中存储了许多用户，每个用户都有一个或多个电子邮件地址。现在，我们得到了属于同一用户的两个电子邮件地址。如果电子邮件地址属于数据库中的不同节点，我们希望删除现有节点，并创建一个新节点，将这两个电子邮件都附加到这个新节点。否则，我们将使用这两个电子邮件创建/更新新的/现有节点。

``` bash
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    # filter is needed to ensure that we do not get same UIDs in u1 and u2
    q1(func: eq(email, "user_email1@company1.io")) @filter(not(eq(email, "user_email2@company1.io"))) {
      u1 as uid
    }

    q2(func: eq(email, "user_email2@company1.io")) @filter(not(eq(email, "user_email1@company1.io"))) {
      u2 as uid
    }

    q3(func: eq(email, "user_email1@company1.io")) @filter(eq(email, "user_email2@company1.io")) {
      u3 as uid
    }
  }

  # case when both emails do not exist
  mutation @if(eq(len(u1), 0) AND eq(len(u2), 0) AND eq(len(u3), 0)) {
    set {
      _:user <name> "user" .
      _:user <dgraph.type> "Person" .
      _:user <email> "user_email1@company1.io" .
      _:user <email> "user_email2@company1.io" .
    }
  }

  # case when email1 exists but email2 does not
  mutation @if(eq(len(u1), 1) AND eq(len(u2), 0) AND eq(len(u3), 0)) {
    set {
      uid(u1) <email> "user_email2@company1.io" .
    }
  }

  # case when email1 does not exist but email2 exists
  mutation @if(eq(len(u1), 0) AND eq(len(u2), 1) AND eq(len(u3), 0)) {
    set {
      uid(u2) <email> "user_email1@company1.io" .
    }
  }

  # case when both emails exist and needs merging
  mutation @if(eq(len(u1), 1) AND eq(len(u2), 1) AND eq(len(u3), 0)) {
    set {
      _:user <name> "user" .
      _:user <dgraph.type> "Person" .
      _:user <email> "user_email1@company1.io" .
      _:user <email> "user_email2@company1.io" .
    }

    delete {
      uid(u1) <name> * .
      uid(u1) <email> * .
      uid(u2) <name> * .
      uid(u2) <email> * .
    }
  }
}' | jq
```

返回结果(当数据库都为空时)：

``` json
{
  "data": {
    "q1": [],
    "q2": [],
    "q3": [],
    "code": "Success",
    "message": "Done",
    "uids": {
      "user": "0x1"
    }
  },
  "extensions": {...}
}
```

返回结果(两个邮件都存在，并附加到不同的节点)：

``` json
{
  "data": {
    "q1": [
      {
        "uid": "0x2"
      }
    ],
    "q2": [
      {
        "uid": "0x3"
      }
    ],
    "q3": [],
    "code": "Success",
    "message": "Done",
    "uids": {
      "user": "0x4"
    }
  },
  "extensions": {...}
}
```

返回结果(当两个电子邮件存在并且已经附加到同一个节点)：

``` json
{
  "data": {
    "q1": [],
    "q2": [],
    "q3": [
      {
        "uid": "0x4"
      }
    ],
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {...}
}
```

也能通过`json`的方式进行相同的操作：

``` bash
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d '{
  "query": "{q1(func: eq(email, \"user_email1@company1.io\")) @filter(not(eq(email, \"user_email2@company1.io\"))) {u1 as uid} \n q2(func: eq(email, \"user_email2@company1.io\")) @filter(not(eq(email, \"user_email1@company1.io\"))) {u2 as uid} \n q3(func: eq(email, \"user_email1@company1.io\")) @filter(eq(email, \"user_email2@company1.io\")) {u3 as uid}}",
  "mutations": [
    {
      "cond": "@if(eq(len(u1), 0) AND eq(len(u2), 0) AND eq(len(u3), 0))",
      "set": [
        {
          "uid": "_:user",
          "name": "user",
          "dgraph.type": "Person"
        },
        {
          "uid": "_:user",
          "email": "user_email1@company1.io",
          "dgraph.type": "Person"
        },
        {
          "uid": "_:user",
          "email": "user_email2@company1.io",
          "dgraph.type": "Person"
        }
      ]
    },
    {
      "cond": "@if(eq(len(u1), 1) AND eq(len(u2), 0) AND eq(len(u3), 0))",
      "set": [
        {
          "uid": "uid(u1)",
          "email": "user_email2@company1.io",
          "dgraph.type": "Person"
        }
      ]
    },
    {
      "cond": "@if(eq(len(u1), 1) AND eq(len(u2), 0) AND eq(len(u3), 0))",
      "set": [
        {
          "uid": "uid(u2)",
          "email": "user_email1@company1.io",
          "dgraph.type": "Person"
        }
      ]
    },
    {
      "cond": "@if(eq(len(u1), 1) AND eq(len(u2), 1) AND eq(len(u3), 0))",
      "set": [
        {
          "uid": "_:user",
          "name": "user",
          "dgraph.type": "Person"
        },
        {
          "uid": "_:user",
          "email": "user_email1@company1.io",
          "dgraph.type": "Person"
        },
        {
          "uid": "_:user",
          "email": "user_email2@company1.io",
          "dgraph.type": "Person"
        }
      ],
      "delete": [
        {
          "uid": "uid(u1)",
          "name": null,
          "email": null
        },
        {
          "uid": "uid(u2)",
          "name": null,
          "email": null
        }
      ]
    }
  ]
}' | jq
```