# Schema 

对于每个谓词，模式指定目标的类型。如果谓词`p`的类型为`T`，那么对于所有主语-谓词-对象三元组`s p o`，对象`o`的模式类型为`T`。

* 在发生突变时，将检查标量类型，如果值不能转换为模式类型，则抛出错误。
* 在查询时，根据谓词的模式类型返回值结果。

如果在突变为谓词添加三元组之前没有指定模式类型，则从第一次突变推断该类型。这种类型是:

* 类型uid，如果谓词的第一个变异具有主语和宾语的节点，或者
* 如果对象是文字且RDF类型出现在第一个变体中，则从RDF类型派生，或者
* 默认类型

## Schema类型

`Dgraph`支持标量类型和`UID`类型。

### Scalar 标量类型

对于所有具有标量类型谓词的三元组，对象都是文字。

`Dgraph`类型 | `Go`类型
----|---
default|string
int|int64
float|float
string|string
bool|bool
dateTime|time.Time(RFC3399 时间格式\[可选的时区\] 例如： 2006-01-02T15:04:05.999999999+10:00 或者 2006-01-02T15:04:05.999999999)
geo|[go-geom](https://github.com/twpayne/go-geom)
password|string(被加密的)

> 注意，只有当dateTime标量类型兼容RFC 3339时，Dgraph才支持日期和时间格式，这与ISO 8601(在RDF规范中定义)不同。在将值发送到Dgraph之前，应该将其转换为RFC 3339格式。

## UID 类型

uid类型表示节点-节点边；在内部，每个节点都表示为一个`uint64 id`。

`Dgraph` 类型|`Go`类型
----|---
uid|uint64

## 添加或者修改 Schema

通过指定模式为列表类型，还可以为标准普尔添加多个标量值。下面示例中的`occupation`可以存储每个`S P`的字符串列表。

索引是用`@index`指定的，用参数指定标记器。当为谓词指定索引时，必须指定索引的类型。例如：

``` schema
name: string @index(exact, fulltext) @count .
multiname: string @lang .
age: int @index(int) .
friend: [uid] @count .
dob: dateTime .
location: geo @index(geo) .
occupations: [string] @index(term) .
```

如果没有为谓词存储数据，模式突变将设置一个空模式，准备接收三元组。

如果数据在突变之前已经存储，则不会检查现有的值以符合新模式。在查询时，Dgraph尝试将现有值转换为新的模式类型，忽略任何转换失败的值。

如果数据存在，并且在模式变化中指定了新索引，则删除未在更新列表中的任何索引，并为每个指定的新标记器创建一个新索引。

如果由模式变异指定，反向边也会被计算。

> 注意，不能定义以dgraph开头的谓词名称。，它保留为Dgraph的内部类型/谓词的名称空间。例如，将dgraph.name定义为谓词是无效的。

## 在后台处理索引

根据数据的大小，索引的计算时间可能会很长。从Dgraph版本20.03.0开始，索引可以在后台计算，因此索引可能在Alter操作返回后仍在运行。这要求您在运行需要新创建索引的查询之前等待索引完成。这样的查询将失败，并出现一个错误，通知给定谓词没有索引或没有反向边。

如果已经在修改一个错误架构，那么alter操作也会失败。请重试。不过，在索引进行时，突变可以成功执行。

例如，假设我们用下面的模式执行一个Alter操作：

``` schema 
name: string @index(fulltext, term) .
age: int @index(int) @upsert .
friend: [uid] @count @reverse .
```

一旦Alter操作返回，Dgraph将报告以下模式，并启动后台任务来计算所有新的索引：

``` schema
name: string .
age: int @upsert .
friend: [uid] .
```

当索引计算完成后，Dgraph将开始报告模式中的索引。在多节点集群中，alpha可能会在不同的时间完成计算索引。在这种情况下，alpha可能会返回不同的模式，直到在所有alpha上完成所有索引的计算。

如果在计算索引时出现意外错误，可能导致后台索引任务失败。您应该重试Alter操作，以便更新架构，或跨所有alpha同步架构。

要了解如何检查后台索引状态，请参见[查询运行状况](https://dgraph.io/docs/master/deploy/dgraph-alpha/#querying-health)。

## HTTP API
*todo*

## Grpc API
*todo*


## Predicate 规则

`Predicate`允许任何字母数字组合。`Dgraph`还支持国际化资源标识符[IRIs](https://en.wikipedia.org/wiki/Internationalized_Resource_Identifier)。您可以在谓词[i18n](https://dgraph.io/docs/query-language/schema/#predicates-i18n)中了解更多信息。

### 允许特殊字符

不接受单个特殊字符，包括来自IRIs的特殊字符。它们必须以字母数字字符作为前缀/后缀：

``` schema
][&*()_-+=!#$%
```

> 注意：我们不会限制您使用@后缀，但是后缀字符将被忽略。

### 被禁止的特殊字符串

``` schema 
^}|{`\~
```

## Predicate 国际化

如果`predicate`是`URI`或具有特定于语言的字符，则在执行模式更改时用尖括号<>将其括起来。

> 注意，`Dgraph`支持谓词名称和值的国际化资源标识符(`IRIs`)。

Schema 解析：

``` schema 
<职业>: string @index(exact) .
<年龄>: int @index(int) .
<地点>: geo @index(geo) .
<公司>: string .
```

该语法允许国际化谓词名称，但全文索引默认仍为英语。要为您的语言使用正确的标记器，您需要使用`@lang`指令并使用语言标记输入值。

Schema:

``` schema 
<公司>: string @index(fulltext) @lang .
```

变更数据：

``` schema
{
  set {
    _:a <公司> "Dgraph Labs Inc"@en .
    _:b <公司> "夏新科技有限责任公司"@zh .
    _:a <dgraph.type> "Company" .
  }
}
```

查询数据：

``` graphql
{
  q(func: alloftext(<公司>@zh, "夏新科技有限责任公司")) {
    uid
    <公司>@.
  }
}
```

## 插入指令 Upsert directive

要在`predicate`上使用`upsert`操作，请在模式中指定`@upsert`指令。当使用`@upsert`指令提交涉及`predicate`的事务时，`Dgraph`会检查索引键是否存在冲突，这有助于在运行并发`upsert`时执行唯一性约束。

这就是为`predicate`指定`upsert`指令的方法。

``` schema 
email: string @index(exact) @upsert .
```

## Noconflict directive

`NoConflict`指令防止在`predicate`级别进行冲突检测。这是一个实验性的特性，不是一个推荐的指令，但它的存在是为了帮助避免不具有高正确性要求的谓词的冲突。这可能会导致数据丢失，特别是当使用带有`count`索引的谓词时。

这就是为谓词指定`@noconflict`指令的方式：

``` schema 
email: string @index(exact) @noconflict .
```

## RDF 类型

`Dgraph`在变更中支持许多`RDF`类型。

除了在第一次更改时暗示模式类型外，`RDF`类型还可以覆盖用于存储的模式类型。

如果`predicate`具有模式类型，而变体具有具有不同基础`Dgraph`类型的`RDF`类型，则会检查模式类型的可转换性，如果它们不兼容，则会抛出错误，但值存储在`RDF`类型对应的`Dgraph`类型中。查询结果总是以模式类型返回。

例如，如果没有为年龄谓词设置模式。给出如下变更：

`Dgraph`:

* 将模式类型设置为`int`，正如前一个三元组所暗示的那样。
* 将`13`在存储上转换为`int`。
* 检查`14`可以转换为`int`，但存储为字符串。
* 为其余两个三元组抛出错误，因为`14.5`不能转换为`int`。

## 扩展类型 Extended Types

### Password 类型

通过将属性的模式设置为`password`类型来设置实体的密码。不能直接查询密码，只能通过`checkpwd`函数进行匹配检查。密码使用`bcrypt`加密。

例如：设置密码，首先设置`schema`，然后设置密码：

``` schema
pass: password .
```

使用变更设置密码：

``` graphql
{
  set {
    <0x123> <name> "Password Example" .
    <0x123> <pass> "ThePassword" .
  }
}
```

测试密码是否正确：

``` graphql
{
  check(func: uid(0x123)) {
    name
    checkpwd(pass, "ThePassword")
  }
}
```

返回：

``` json 
{
  "data": {
    "check": [
      {
        "name": "Password Example",
        "checkpwd(pass)": true
      }
    ]
  }
}
```

在`checkpwd`函数上使用别名：

``` graphql
{
  check(func: uid(0x123)) {
    name
    secret: checkpwd(pass, "ThePassword")
  }
}
```

返回：

``` json 
{
  "data": {
    "check": [
      {
        "name": "Password Example",
        "secret": true
      }
    ]
  }
}
```

### Indexing 索引

> 在一个 predicate 上使用函数需要指定索引

当通过应用函数进行筛选时，`Dgraph`使用索引来高效地搜索可能较大的数据集。

所有标量类型都可以被索引。

类型`int`, `float`, `bool`和`geo`都只有一个默认索引：带有命名为`int`, `float`, `bool`和`geo`的标记器。

类型`string`和`dateTime`有许多索引。


#### String Indices

`String`类型的值可用下表的索引进行检索：

`Dgraph`函数|需要的索引或`tokenizer`|标注
----|----|---
`eq`|`hash` `exact` `term` `fulltext`|eq的最佳性能指标是哈希值。只有在您还需要术语或全文搜索时才使用`term`或`fulltext`。如果您已经在使用`term`，那么也不需要使用`hash`或`exact`。
`le` `ge` `lt` `gt` | `exact` |允许更快地查询
`allofterms` `amyofterms`| `term`|允许在一个句子中使用短语查询
`alloftext` `anyoftext`| `fulltext` | 匹配特定语言的词干和停止词。
`regexp`| `trigram`|正则表达式匹配。也可用于相等检查。

> 错误的索引选择可能会导致性能损失和增加事务冲突率。只使用应用程序需要的最小数量和最简单的索引。

#### DateTime Indices

`DateTime`类型的值可用下表地索引进行检索：

索引名字 `tokenizer`|索引到的日期的部分
----|----
`year`|索引到年份(默认)
`month`|索引年份和月份
`day`|索引年份、月份、天
`hour`|索引年份、月份、天、小时

`dateTime`索引的选择允许选择索引的精度。应用程序，如这些文档中的电影示例，需要搜索日期，但每年的节点相对较少，可能更喜欢年份标记器;依赖于细粒度日期搜索(如实时传感器读数)的应用程序可能更喜欢小时索引。

所有的`dateTime`索引都是可排序的。


#### Sortable Indices

并不是所有的索引都在它们索引的值之间建立一个总的顺序。可排序索引允许不等式函数和排序。

* 索引`int`和`float`是可排序的。
* 字符串索引精确是可排序的。
* 所有的`dateTime`索引都是可排序的。

例如，给定一个字符串类型的边名，要按名称排序或对名称执行不等过滤，必须指定确切的索引。在这种情况下，模式查询将至少返回以下标记器：

``` json 
{
  "predicate": "name",
  "type": "string",
  "index": true,
  "tokenizer": [
    "exact"
  ]
}
```

#### Count Indices

对于具有`@count` 的`predicate`，将每个节点的边数索引为索引。这样可以快速查询表单：

``` graphql
{
  q(func: gt(count(pred), threshold)) {
    ...
  }
}
```


### List 类型

如果在Schema中指定，变量类型的`predicate`也可以存储值的`list`。标量类型需要包含在`[]`中，以表明它是一个`list`类型。

``` schema 
occupations: [string] .
score: [int] .
```

* `set`操作添加到值列表中。存储值的顺序是不确定的。
* `delete`操作从列表中删除值。
* 查询这些谓词将返回数组中的列表。
* 索引可以应用于具有列表类型的谓词，并且可以对其使用函数。
* 不允许使用这些谓词进行排序。
* 这些列表就像一个无序的集合。例如：["e1"， "e1"， "e2"]可能会被存储为["e2"， "e1"]，也就是说，重复的值将不会被存储，顺序也不会被保留。

#### 在List中使用过滤函数

`Dgraph`支持基于`list`的过滤。过滤的工作原理类似于它在`edge`上的工作原理，并且具有相同的可用函数。

例如，查询或父边的`@filter(eq(occupation，"Teacher"))`将显示数组中每个节点的列表中的所有职业，但只包含将`Teacher`作为职业之一的节点。但是，不支持值边过滤。

### 反转一条边

图的边是单向的。对于节点-节点边，有时建模需要反向边。如果只有一些主-谓词-对象三元组有反向，则必须手动添加它们。但是如果一个谓词总是有反向，如果在模式中指定了`@reverse`，则`Dgraph`会计算反向边。

`anEdge`的反向边是`~anEdge`。

对于现有数据，`Dgraph`计算所有反向边。对于schema变更后添加的数据，`Dgraph`计算并存储每个添加的三元组的反向边。

``` schema
type Person {
  name string
}
type Car {
  regnbr string
  owner Person
}
owner uid @reverse .
regnbr string @index(exact) .
name string @index(exact) .
```

这使得查询`person`和他们的车成为可能：

``` graphql
q(func type(Person)) {
  name
  ~owner { name }
}
```

为了在结果中得到一个不同于`~owner`的键，可以用想要的标签(在本例中是`cars`)来编写查询：

``` graphql
q(func type(Person)) {
  name
  cars: ~owner { name }
}
```

如果一辆车有多个“所有者”，这也适用：

``` schema
owner [uid] @reverse .

```

在这两种情况下，`owner`边都应该设置在`Car`上：

``` schema
_:p1 <name> "Mary" .
_:p1 <dgraph.type> "Person" .
_:c1 <regnbr> "ABC123" .
_:c1 <dgraph.type> "Car" .
_:c1 <owner> _:p1
```


### 查询 Schema

你可以使用下面的`DQL`语句查询整个`Schema`： 

``` dql
schema {}
```

> 注意：与常规查询不同，模式查询没有被花括号包围。此外，模式查询和常规查询不能一起使用。


您可以在查询体中查询特定的模式字段：

``` dql
schema {
  type
  index
  reverse
  tokenizer
  list
  count
  upsert
  lang
}
```

你也可以查询特定的谓词：

``` dql
schema(pred: [name, friend]) {
  type
  index
  reverse
  tokenizer
  list
  count
  upsert
  lang
}
```

> 注意，如果启用了ACL，那么模式查询只返回登录的ACL用户具有读访问权限的谓词。

类型也可以查询。下面是一些示例查询：

``` dql
schema(type: Movie) {}
schema(type: [Person, Animal]) {}
```


注意，类型查询不包含花括号之间的任何内容。输出将是请求类型的完整定义。

