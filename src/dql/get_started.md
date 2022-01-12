# 开始

> 这是一份快速开始的手册，你可以在[这里](https://dgraph.io/docs/tutorials/)找到开始系列的教程。

## Dgraph

`Dgraph`从头开始设计，以便在生产环境中运行，它是带有`GraphQL`后端的原生`GraphQL`数据库。它是开源的，可扩展的，分布式的，高可用性的和具有闪电般的速度。


`Dgraph`集群由不同的节点(`Zero`, `Alpha`和`Ratel`)组成，每个节点都有不同的用途：

* `Dgraph Zero`控制`Dgraph`集群，将服务器分配给一个组，并在服务器组之间重新平衡数据。
* `Dgraph Alpha`托管谓词(predicate)和索引(index)。谓词是与节点(node)关联的属性(facets)或两个节点之间的关系(relation)。索引是可以与谓词关联的标记器，可以使用适当的函数启用过滤。
* `Ratel`服务于`UI`来运行查询、变更和更改`Schema`。


你至少需要一个`Dgraph Zero`和一个`Dgraph Alpha`来启动`Dgraph`数据库。

**这里有一个四个步骤的教程来帮助你启动和运行。**

这是一个运行`Dgraph`的快速入门指南。如果你想了解互动的攻略，那就来看看吧。

> 提示：本指南针对的是强大的`Dgraph`查询语言，`DQL`是`Facebook`创建的一种查询语言`GraphQL`的变体。您可以从`dgraph.io/ GraphQL`中找到开始使用`GraphQL`的说明。

## 步骤一：运行 Dgraph

安装和运行`Dgraph`有几种方法，都可以在下载页面中找到。

让`Dgraph`启动并运行的最简单方法是使用`dgraph/standalone`这个`docker`映像。如果您还没有`Docker`，请按照这里的说明安装它：

``` bash
docker run --rm -it -p "8080:8080" -p "9080:9080" -p "8000:8000" -v ~/dgraph:/dgraph "dgraph/standalone:v21.03.2"
```

> 注意：此独立映像仅用于快速启动目的。不建议在生产环境中使用。

这将启动一个容器，其中运行`Dgraph Alpha`、`Dgraph Zero`和`Ratel`。您会发现`Dgraph`数据存储在主目录的一个名为`Dgraph`的文件夹中。

## 使用变更

> 提示：`Dgraph`运行起来后，您可以通过`http://localhost:8000`访问`Ratel`。它允许基于浏览器的查询、变化和可视化。您可以通过命令行中的curl或将突变数据粘贴到`Ratel`中来运行下面的突变和查询。

### 数据集

数据集是一个电影图，其中的图节点是类型导演、演员、类型或电影的实体。

### 在 `Dgraph` 中检索数据

更改存储在`Dgraph`中的数据叫做`变更`(mutation)。到目前为止，`Dgraph`支持通过两种方式变更数据：`RDF`和`JSON`。下面的`RDF`变更存储关于`星球大战`系列的前三次发行和`星际迷航`系列电影之一的信息。提交`RDF`变更(通过`curl`或`Ratel UI`的`mutate`选项卡)将在`Dgraph`中存储数据：

``` bash
curl "localhost:8080/mutate?commitNow=true" --silent --request POST \
 --header  "Content-Type: application/rdf" \
 --data $'
{
  set {
   _:luke <name> "Luke Skywalker" .
   _:luke <dgraph.type> "Person" .
   _:leia <name> "Princess Leia" .
   _:leia <dgraph.type> "Person" .
   _:han <name> "Han Solo" .
   _:han <dgraph.type> "Person" .
   _:lucas <name> "George Lucas" .
   _:lucas <dgraph.type> "Person" .
   _:irvin <name> "Irvin Kernshner" .
   _:irvin <dgraph.type> "Person" .
   _:richard <name> "Richard Marquand" .
   _:richard <dgraph.type> "Person" .

   _:sw1 <name> "Star Wars: Episode IV - A New Hope" .
   _:sw1 <release_date> "1977-05-25" .
   _:sw1 <revenue> "775000000" .
   _:sw1 <running_time> "121" .
   _:sw1 <starring> _:luke .
   _:sw1 <starring> _:leia .
   _:sw1 <starring> _:han .
   _:sw1 <director> _:lucas .
   _:sw1 <dgraph.type> "Film" .

   _:sw2 <name> "Star Wars: Episode V - The Empire Strikes Back" .
   _:sw2 <release_date> "1980-05-21" .
   _:sw2 <revenue> "534000000" .
   _:sw2 <running_time> "124" .
   _:sw2 <starring> _:luke .
   _:sw2 <starring> _:leia .
   _:sw2 <starring> _:han .
   _:sw2 <director> _:irvin .
   _:sw2 <dgraph.type> "Film" .

   _:sw3 <name> "Star Wars: Episode VI - Return of the Jedi" .
   _:sw3 <release_date> "1983-05-25" .
   _:sw3 <revenue> "572000000" .
   _:sw3 <running_time> "131" .
   _:sw3 <starring> _:luke .
   _:sw3 <starring> _:leia .
   _:sw3 <starring> _:han .
   _:sw3 <director> _:richard .
   _:sw3 <dgraph.type> "Film" .

   _:st1 <name> "Star Trek: The Motion Picture" .
   _:st1 <release_date> "1979-12-07" .
   _:st1 <revenue> "139000000" .
   _:st1 <running_time> "132" .
   _:st1 <dgraph.type> "Film" .
  }
}
' | python -m json.tool | less
```

> 提示：要通过`curl`使用文件提交`RDF/JSON`变更，可以使用`curl`选项`--data-binary @/path/ To /mutation`。`--data-binary`选项跳过`curl`的默认`url`编码，包括删除所有换行符。因此，通过使用`--data-binary`选项，您可以在文本中使用`#`注释，因为使用`--data`选项，文本中第一个`#`之后的任何内容都将出现在同一行，因此将被视为一个长注释。

### 步骤三：修改Schema

修改`Schema`，在某些数据上添加索引，这样查询就可以使用术语匹配、过滤和排序。

``` bash
curl "localhost:8080/alter" --silent --request POST \
  --data $'
name: string @index(term) .
release_date: datetime @index(year) .
revenue: float .
running_time: int .
starring: [uid] .
director: [uid] .

type Person {
  name
}

type Film {
  name
  release_date
  revenue
  running_time
  starring
  director
}
' | python -m json.tool | less
```

> 提示：要从`Ratel UI`提交Schema``，请转到`Schema`页面，单击批量编辑，然后粘贴`Schema`。

### 步骤四：运行查询

#### 查询所有电影

运行此查询以获取所有电影。该查询列出了所有具有主演优势的电影：

> 提示：您也可以在`Ratel UI`的`query`选项卡中运行`DQL`查询。

``` bash
curl "localhost:8080/query" --silent --request POST \
  --header "Content-Type: application/dql" \
  --data $'
{
 me(func: has(starring)) {
   name
  }
}
' | python -m json.tool | less
```

> 注意GraphQL+-已重命名为Dgraph查询语言(DQL)。虽然application/dql是Content-Type头的首选值，但我们将继续支持Content-Type: application/graphql+-，以避免进行破坏性的更改。

#### 获得1980年之后发行的所有电影

运行这个查询可以得到“1980年”之后发行的“星球大战”电影。在用户界面中尝试它，以图形的形式查看结果：

``` bash
curl "localhost:8080/query" --silent --request POST \
  --header "Content-Type: application/dql" \
  --data $'
{
  me(func: allofterms(name, "Star Wars"), orderasc: release_date) @filter(ge(release_date, "1980")) {
    name
    release_date
    revenue
    running_time
    director {
     name
    }
    starring (orderasc: name) {
     name
    }
  }
}
' | python -m json.tool | less
```

返回结果：

``` json
{
  "data":{
    "me":[
      {
        "name":"Star Wars: Episode V - The Empire Strikes Back",
        "release_date":"1980-05-21T00:00:00Z",
        "revenue":534000000.0,
        "running_time":124,
        "director":[
          {
            "name":"Irvin Kernshner"
          }
        ],
        "starring":[
          {
            "name":"Han Solo"
          },
          {
            "name":"Luke Skywalker"
          },
          {
            "name":"Princess Leia"
          }
        ]
      },
      {
        "name":"Star Wars: Episode VI - Return of the Jedi",
        "release_date":"1983-05-25T00:00:00Z",
        "revenue":572000000.0,
        "running_time":131,
        "director":[
          {
            "name":"Richard Marquand"
          }
        ],
        "starring":[
          {
            "name":"Han Solo"
          },
          {
            "name":"Luke Skywalker"
          },
          {
            "name":"Princess Leia"
          }
        ]
      }
    ]
  }
}
```

就是这样！在这四个步骤中，我们设置了`Dgraph`、添加了一些数据、设置了一个`Schema`并查询了该数据。

**下一步该做什么**

* 转到Clients查看如何从应用程序与Dgraph通信。
* 请参阅本教程，了解如何在Dgraph中编写查询。
* 查询语言参考中还可以找到更广泛的查询。
* 如果希望在集群中运行Dgraph，请参阅部署。