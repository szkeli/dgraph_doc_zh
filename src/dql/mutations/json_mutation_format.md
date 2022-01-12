# JSON 变更格式 (JSON Mutation Format)

你也能够通过`JSON`对象指定一个变更。这可以让变更以更自然的方式表达出来。它还消除了应用程序自定义序列化代码的需要，因为大多数语言已经有了`JSON`处理库。

当`Dgraph`接收到`JSON`对象的变更时，它首先将其转换为内部边格式，然后再将其处理为`Dgraph`。

**对于`JSON`：**

JSON -> Edges -> Posting list

**对于`RDF`：**

RDF -> Edges -> Posting list

每个JSON对象表示图中的单个节点。

> 注意：`JSON`变更可以通过`Go`客户端、`JavaScript`客户端和`Java`客户端等`gRPC`客户端使用，也可以通过`dgraph-js-http`和`cURL`对`HTTP`客户端使用。在[这里](https://dgraph.io/docs/mutations/json-mutation-format/#using-json-operations-via-curl)可以看到更多关于`cURL`的信息。

## 设置字面量值

当设置新值时，变更消息中的`set_json`字段应该包含一个`JSON`对象。

字面量值可以直接通过向`JSON`对象设置`key/value`来进行。`key`将表示谓词(predicate)，`value`将表示对象。

例如：

``` json 
{
  "name": "diggy",
  "food": "pizza",
  "dgraph.type": "Mascot"
}
```

上面`JSON`对象将被转化为如下`RDF`：

``` rdf
_:blank-0 <name> "diggy" .
_:blank-0 <food> "pizza" .
_:blank-0 <dgraph.type> "Mascot" .
```


变更的返回结果是一个`Map`，该`Map`具有一个`uid`，这个`uid`对应于键`blank-0`。你也可以像下面这样指定你自己的`uid`键：

``` json 
{
  "uid": "_:diggy",
  "name": "diggy",
  "food": "pizza",
  "dgraph.type": "Mascot"
}
```

在这种情况下，分配的`uid`映射将有一个名为`diggy`的键，其值是分配给它的`uid`。

### 禁止值 (Forbidden values)

> 使用`JSON`提交变更时，`uid()` 和 `val()` 这两个字符串是不允许使用的。

例如：`Dgraph`不能处理以下提交的变更：

``` json 
{
  "uid": "uid(t)",
  "user_name": "uid(s ia)",
  "name": "val (s kl)",
}
```

## 多语言支持 (Language Support)

`RDF`变更和`JSON`变更之间的一个重要区别在于指定字符串值的语言。在`JSON`中，语言标记被附加到边名，而不是像`RDF`那样的值。

例如，下面是一个使用`JSON`提交的变更：

``` json 
{
  "food": "taco",
  "rating@en": "tastes good",
  "rating@es": "sabe bien",
  "rating@fr": "c'est bon",
  "rating@it": "è buono",
  "dgraph.type": "Food"
}
```

作为对比，使用`RDF`提交相同的变更需要：

``` rdf
_:blank-0 <food> "taco" .
_:blank-0 <dgraph.type> "Food" .
_:blank-0 <rating> "tastes good"@en .
_:blank-0 <rating> "sabe bien"@es .
_:blank-0 <rating> "c'est bon"@fr .
_:blank-0 <rating> "è buono"@it .
```

## 地理位置支持 (Geolocation support)

`JSON`变更支持地理位置数据。地理位置数据以`JSON`对象的形式输入，带有`type`和`coordinates`键。记住，我们只支持在`Point`、`Polygon`和`MultiPolygon`类型上建立索引，但是我们可以存储其他类型的地理位置数据。下面是一个例子：

``` json 
{
  "food": "taco",
  "location": {
    "type": "Point",
    "coordinates": [1.0, 2.0]
  }
}
```

## 引用已存在的节点 (Referencing existing nodes)

如果`JSON`对象包含一个名为`uid`的字段，那么该字段将被解释为图中现有节点的`uid`。这种机制允许您引用现有的节点。

例如：

``` json 
{
  "uid": "0x467ba0",
  "food": "taco",
  "rating": "tastes good",
  "dgraph.type": "Food"
}
```

上面的`JSON`对象将被转化为以下`RDF`：

``` rdf
<0x467ba0> <food> "taco" .
<0x467ba0> <rating> "tastes good" .
<0x467ba0> <dgraph.type> "Food" .
```

## 节点之间的边 (Edges between nodes)

节点之间的边以类似于字面量值的方式表示，只是对象是一个`JSON`对象：

``` json 
{
  "name": "Alice",
  "friend": {
    "name": "Betty"
  }
}
```

上面的`JSON`对象将会被转化为：

``` rdf
_:blank-0 <name> "Alice" .
_:blank-0 <friend> _:blank-1 .
_:blank-1 <name> "Betty" .
```

变更的结果将包含分配给`blank-0`和`blank-1`节点的`uid`。如果您想在不同的键下返回这些`uid`，您可以将`uid`字段指定为一个空白节点：

``` json
{
  "uid": "_:alice",
  "name": "Alice",
  "friend": {
    "uid": "_:bob",
    "name": "Betty"
  }
}
```

上面的`JSON`对象将会被转化为：

``` rdf
_:alice <name> "Alice" .
_:alice <friend> _:bob .
_:bob <name> "Betty" .
```

可以使用与添加字面量值相同的方式引用现有节点。例如连接两个现有节点：

``` json 
{
  "uid": "0x123",
  "link": {
    "uid": "0x456"
  }
}
```

将会被转化为：

``` rdf
<0x123> <link> <0x456> .
```

> 注意：一个常见的错误是尝试使用`{"uid":"0x123"，"link":"0x456"}`。这将导致一个错误：`Dgraph`将这个`JSON`对象解释为将链接谓词设置为字符串`0x456`，这通常不是你希望的。


## 删除字面量值 (Deleting literal values)

`JSON`也能提交删除变更。

要发送删除变更，在变更消息中使用`delete_json`字段而不是`set_json`字段。

> 注意：如果您使用`dgraph-js-http`客户端或`Ratel UI`，请检查`JSON`语法使用原始`HTTP`或`Ratel UI`部分。

在使用删除变更时，始终必须引用现有节点。因此，必须提供每个`JSON`对象的`uid`字段。应该删除的谓词应该设置为`JSON`值`null`。

例如，要删除食物评级：

``` json 
{
  "uid": "0x467ba0",
  "rating": null
}
```

## 删除边

删除一条边需要创建该边的相同`JSON`对象。例如，删除谓词链接`0x123`到`0x456`：

``` json 
{
  "uid": "0x123",
  "link": {
    "uid": "0x456"
  }
}
```

从单个节点发出的谓词(predicate)的所有边都可以一次删除(相当于删除`S P *`)：

``` json 
{
  "uid": "0x123",
  "link": null
}
```

如果没有指定谓词，则删除节点的所有已知出站边(到其他节点和到字面量值)(对应于删除`S * *`)。要删除的谓词是使用类型系统派生的。有关更多信息，请参阅[RDF格式文档](https://dgraph.io/docs/mutations/delete/)和[类型系统](https://dgraph.io/docs/query-language/type-system/)部分：

``` json 
{
  "uid": "0x123"
}
```

## Facets

可以通过使用`|`字符来分隔`JSON`对象字段名中的谓词和`facet`键来创建`facet`。这与用于在查询结果中显示`facet`的编码模式相同。如：

``` json 
{
  "name": "Carol",
  "name|initial": "C",
  "dgraph.type": "Person",
  "friend": {
    "name": "Daryl",
    "friend|close": "yes",
    "dgraph.type": "Person"
  }
}
```

等价于下面的`RDF`：


``` rdf
_:blank-0 <name> "Carol" (initial="C") .
_:blank-0 <dgraph.type> "Person" .
_:blank-0 <friend> _:blank-1 (close="yes") .
_:blank-1 <name> "Daryl" .
_:blank-1 <dgraph.type> "Person" .
```

`facet`不包含类型信息，但是`Dgraph`将尝试从输入猜测类型。如果一个`facet`的值可以被解析为一个数字，那么它将被转换为`float`或`int`。如果它可以被解析为一个布尔值，那么它将被存储为一个布尔值。如果值是字符串，如果该字符串与`Dgraph`能识别的时间格式(`YYYY`, `MM-YYYY`, `DD-MM-YYYY`, `RFC339`等)中的一种相匹配，那么它将被存储为`datetime`，否则将被存储为双引号字符串。如果不想冒`facet`数据被误读为时间值的风险，最好将数值数据存储为`int`或`float`类型。

## 删除 Facets

删除`Facet`最简单的方法是覆盖它。当您为没有`facet`的相同实体创建一个新的变更时，现有的`facet`将被自动删除：

``` rdf
<0x1> <name> "Carol" .
<0x1> <friend> <0x2> .
```

另一种方法是使用`Upsert Block`。

在下面的查询中，我们将删除`Name`和`Friend`谓词中的`Facet`。要覆盖，我们需要收集执行此操作的边的值，并使用`val(var)`函数来完成覆盖：

``` bash 
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    user as var(func: eq(name, "Carol")){
      Name as name
      Friends as friend
    }
  }

  mutation {
    set {
      uid(user) <name> val(Name) .
      uid(user) <friend> uid(Friends) .
    }
  }
}' | jq
```

## 使用 `JSON`创建一个`List`然后对这个`List`操作：

Schema:

``` schema
testList: [string] .
```

在`testList`中添加一些数据：

``` json 
{
  "testList": [
    "Grape",
    "Apple",
    "Strawberry",
    "Banana",
    "watermelon"
  ]
}
```

现在让我们把`Apple`从这个列表中删除(注意，`List`中的项是区分大小写的)：

``` dql
{
  q(func: has(testList)) {
    uid
    testList
  }
}
```

``` json 
{
  "delete": {
    "uid": "0x6", #UID of the list.
    "testList": "Apple"
  }
}
```

当然，你也能够一次性删除多个值：

``` json 
{
  "delete": {
    "uid": "0x6",
    "testList": [
      "Strawberry",
      "Banana",
      "watermelon"
    ]
  }
}
```

> 注意：如果您使用`dgraph-js-http`客户端或`Ratel UI`，请检查`JSON`语法使用原始`HTTP`或`Ratel UI`部分。

添加一个水果：

``` json 
{
   "uid": "0x6", #UID of the list.
   "testList": "Pineapple"
}
```

## JSON 与 List 类型的 facet

schema: 

``` rdf
<name>: string @index(exact).
<nickname>: [string] .
```

要创建`list`类型的谓词(predicate)，需要指定单个列表中的所有值。所有谓词值的`facet`应该一起指定。它是用`map`格式完成的，列表中的谓词值的索引是`map`键，它们各自的`facet`值是`map`值。不包含`facet`值的谓词值将从`facet`映射中丢失。如：

``` json 
{
  "set": [
    {
      "uid": "_:Julian",
      "name": "Julian",
      "nickname": ["Jay-Jay", "Jules", "JB"],
      "nickname|kind": {
        "0": "first",
        "1": "official",
        "2": "CS-GO"
      }
    }
  ]
}
```

在上面您可以看到，我们有三个值，它们各自具有方面，可以进入列表。你可以运行这个查询来检查包含`facet`的列表：

``` dql
{
  q(func: eq(name,"Julian")) {
  uid
  nickname @facets
  }
}
```

之后如果希望使用`facet`添加更多值，只需执行相同的过程，但现在必须使用相应节点的`UID`，而不是使用空白节点：

``` json 
{
  "set": [
    {
      "uid": "0x3",
      "nickname|kind": "Internet",
      "nickname": "@JJ"
    }
  ]
}
```

最后返回的结果：

``` json 
{
  "data": {
    "q": [
      {
        "uid": "0x3",
        "nickname|kind": {
          "0": "first",
          "1": "Internet",
          "2": "official",
          "3": "CS-GO"
        },
        "nickname": [
          "Jay-Jay",
          "@JJ",
          "Jules",
          "JB"
        ]
      }
    ]
  }
}
```

## 指定多个操作

当指定添加或删除变更时，可以使用`JSON`数组同时指定多个节点。

例如，下面的`JSON`对象可以用来添加两个新节点，每个节点都有一个名称：

``` json 
[
  {
    "name": "Edward"
  },
  {
    "name": "Fredric"
  }
]
```

## 使用 `Raw HTTP` 或 `Ratel UI` 解析 `JSON`

这种语法可以在最新版本的`Ratel`中使用，也可以在`dgraph-js-http`客户端中使用，甚至可以通过`cURL`使用。

您也可以下载`Ratel UI`。

变更：

``` dql
{
  "set": [
    {
      # One JSON obj in here
    },
    {
      # Another JSON obj in here for multiple operations
    }
  ]
}
```

删除：
删除操作与删除字面量值和删除边相同。

``` dql
{
  "delete": [
    {
      # One JSON obj in here
    },
    {
      # Another JSON obj in here for multiple operations
    }
  ]
}
```

## 通过 `cURL` 使用 `JSON` 操作

首先，你需要设置 `HTTP` 相应的请求头为 `application/json`

``` bash
-H 'Content-Type: application/json'
```

> 注意：为了使用`jq`进行`JSON`格式化，你需要`jq`包。有关安装细节，请参阅jq下载页面。你也可以使用`Python`的内置`json`工具模块，使用`python -m json`。

``` bash 
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d $'
    {
      "set": [
        {
          "name": "Alice"
        },
        {
          "name": "Bob"
        }
      ]
    }' | jq

```

删除操作：

``` bash
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d $'
    {
      "delete": [
        {
          "uid": "0xa"
        }
      ]
    }' | jq
``` 

或者你也可以提交一个在文件中的变更：

``` bash 
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d @data.json
```

`data.json` 中文件内容类似于：

``` json
{
  "set": [
    {
      "name": "Alice"
    },
    {
      "name": "Bob"
    }
  ]
}
```

对于`HTTP`上的变更，`JSON`文件必须遵循相同的格式：一个带有`set`或`delete`键的`JSON`对象和一个用于变更的`JSON`对象数组。如果您已经拥有一个包含数据数组的文件，则可以使用`jq`将数据转换为适当的格式。例如，如果你的`Json`文件看起来像这样：

``` json 
[
  {
    "name": "Alice"
  },
  {
    "name": "Bob"
  }
]
```

然后可以使用以下`jq`命令将数据转换为适当的格式，其中`.`在`jq '{set: .}'`中表示`data.json`的内容：

``` bash
cat data.json | jq '{set: .}'
```

``` json 
{
  "set": [
    {
      "name": "Alice"
    },
    {
      "name": "Bob"
    }
  ]
}
```