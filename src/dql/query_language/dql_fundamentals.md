# DQL 基础

Dgraph Query Language, DQL查询语言（旧称GraphQL+-），是一门基于`GraphQL`的查询语言。`GraphQL`不是专门为图数据库而设计的，但是它的解析规则和图查询非常相似（查询语法的解析，schema的校验，子图塑型返回等），所以我们就直接在`GraphQL`的基础上增删一些特性，创造了`DQL`。

`DQL`目前还在开发中，我们在未来可以会增加一些新特性，以及简化目前的版本。

## 一个小示例

这个小例子使用了一个包含2.1千万条元组的电影和戏子的数据库。负载这个数据库的`Dgraph`是跑在云[https://play.dgraph.io/](https://play.dgraph.io/)上面的。你也可以选择[本地运行](../get_started.md)

### 定义数据库的数据结构
本例子使用下面的`schema`定义数据库的结构:

``` schema
# Define Directives and index

director.film: [uid] @reverse .
actor.film: [uid] @count .
genre: [uid] @reverse .
initial_release_date: dateTime @index(year) .
name: string @index(exact, term) @lang .
starring: [uid] .
performance.film: [uid] .
performance.character_note: string .
performance.character: [uid] .
performance.actor: [uid] .
performance.special_performance_type: [uid] .
type: [uid] .

# Define Types

type Person {
  name
  director.film
  actor.film
}

type Movie {
  name
  initial_release_date
  genre
  starring
}

type Genre {
  name
}

type Performance {
  performance.film
  performance.character
  performance.actor
}
```

## 开始查询

对一些`node`进行的查询是建立在图数据库的搜索规则，匹配模式上的，返回一个图作为查询结果。

一个查询是由几个查询块组成的：先由一个根查询块查询到一系列符合查询规则的`nodes`，然后再在这些`nodes`上面应用图匹配和图过滤。

> 更多的查询列子可以参照[查询设计原则](../design_concepts/concepts.md)

### 错误码

当你使用`DQL`查询的时候，服务端可能会返回一个查询错误。在服务端返回的错误对象`JSON`中有一个`code`属性，`code`通常有两个取值：

1. `ErrorInvalidRequest`: 可能是错误的请求（400）或者是服务器内部的错误（500）
2. `Error`: 服务器内部的错误（500）

例如，当你提交请求解析错误时，服务端返回:

``` json
{
  "errors": [
    {
      "message": "while lexing {\nq(func: has(\"test)){\nuid\n}\n} at line 2 column 12: Unexpected end of input.",
      "extensions": {
        "code": "ErrorInvalidRequest"
      }
    }
  ],
  "data": null
}
```

#### Error

这是一个很少见的错误，一般是服务器序列化`GO`struct到`JSON`对象时发生错误导致

#### ErrorInvalidRequst

### 返回值

每一个查询都有一个名字，和你发送给后端的查询的位于根块的名字一致。
如果一条边是一个值类型，那么这条边将直接返回。
在本例子的数据库中，边（`edges`）连接电影`movies`到导演`directors`和戏子`actors`，`movies`拥有`name`，`release date`更新时间，还有它在出名的影视数据库的`id`。

下面是一个名为`bladerunner`的请求：通过比对电影的名字`Blade Runner`获取电影信息：

``` dql
{
  bladerunner(func: eq(name@en, "Blade Runner")) {
    uid
    name@en
    initial_release_date
    netflix_id
  }
}
```

它将会返回：

``` json
{
  "data": {
    "bladerunner": [
      {
        "uid": "0x394c",
        "name@en": "Blade Runner",
        "initial_release_date": "1982-06-25T00:00:00Z",
        "netflix_id": "70083726"
      }
    ]
  }
}
```

上面的请求首先通过索引找到`name`边等于`Blade Runner`的`node`，然后返回该`node`的出边属性。每一个`node`都有一个唯一的64位的`id`，上面的请求中，`uid`边返回了该`node`的`id`，如果一个`node`的`id`已知，可以通过`uid`函数直接查找该节点。

``` dql
{
  bladerunner(func: uid(0x394c)) {
    uid
    name@en
    initial_release_date
    netflix_id
  }
}
```

上面的请求将返回：

``` json
{
  "data": {
    "bladerunner": [
      {
        "uid": "0x394c",
        "name@en": "Blade Runner",
        "initial_release_date": "1982-06-25T00:00:00Z",
        "netflix_id": "70083726"
      }
    ]
  }
}
```

一个查询可能匹配到多个`node`，下面查询所有名字含有`Blade`或`Runner`的节点：

``` dql
{
  bladerunner(func: anyofterms(name@en, "Blade Runner")) {
    uid
    name@en
    initial_release_date
    netflix_id
  }
}
```

``` json
{
  "data": {
    "bladerunner": [
      {
        "uid": "0xd5f",
        "name@en": "The Runner"
      },
      {
        "uid": "0x394c",
        "name@en": "Blade Runner",
        "initial_release_date": "1982-06-25T00:00:00Z",
        "netflix_id": "70083726"
      },
      ....
    ]
  }
}
```

`uid`函数也能同时指定多个`id`：

``` dql
{
  movies(func: uid(0xb5849, 0x394c)) {
    uid
    name@en
    initial_release_date
    netflix_id
  }
}
```

返回
``` json 
{
  "data": {
    "movies": [
      {
        "uid": "0x394c",
        "name@en": "Blade Runner",
        "initial_release_date": "1982-06-25T00:00:00Z",
        "netflix_id": "70083726"
      },
      {
        "uid": "0xb5849",
        "name@en": "James Cameron's Explorers: From the Titanic to the Moon",
        "initial_release_date": "2006-01-01T00:00:00Z",
        "netflix_id": "70058064"
      }
    ]
  }
}
```

## 展开图节点的边

一个查询能通过嵌套查询块`{}`从一个节点扩展到另一个节点
下面查询`Blade Runner`中的演员和人物：该查询首先查询出拥有名字为`Blade Runner`这条边的节点，然后从这个节点沿着向外的主演边`starring`找到演员饰演的角色的节点。从这些节点里面再沿着`perfromance.actor`边和`performance.character`边展开，找到演员的名字和在剧中饰演的角色的名字。

``` dql
{
  brCharacters(func: eq(name@en, "Blade Runner")) {
    name@en
    initial_release_date
    starring {
      performance.actor {
        name@en  # actor name
      }
      performance.character {
        name@en  # character name
      }
    }
  }
}
```

``` json 
{
  "data": {
    "brCharacters": [
      {
        "name@en": "Blade Runner",
        "initial_release_date": "1982-06-25T00:00:00Z",
        "starring": [
          {
            "performance.actor": [
              {
                "name@en": "John Edward Allen"
              }
            ],
            "performance.character": [
              {
                "name@en": "Kaiser"
              }
            ]
          },
          {
            "performance.actor": [
              {
                "name@en": "Joanna Cassidy"
              }
            ],
            "performance.character": [
              {
                "name@en": "Zhora"
              }
            ]
          }
          ...
        ]
      }
    ]
  }
}
```


## 注释

所有在 `#`后面的都是注释

## 使用过滤

查询根查找一组初始节点，查询通过返回值并沿着边继续查询————查询中到达的任何节点都是在根处搜索之后遍历找到的。找到的节点可以通过应用`@filter`过滤，可以在根节点之后或任何边进行过滤。

查询示例:《银翼杀手》导演`Ridley Scott`在2000年之前发行的电影：

``` dql
{
  scott(func: eq(name@en, "Ridley Scott")) {
    name@en
    initial_release_date
    director.film @filter(le(initial_release_date, "2000")) {
      name@en
      initial_release_date
    }
  }
}
```

将返回：

``` json 
{
  "data": {
    "scott": [
      {
        "name@en": "Ridley Scott",
        "director.film": [
          {
            "name@en": "Blade Runner",
            "initial_release_date": "1982-06-25T00:00:00Z"
          },
          {
            "name@en": "Alien",
            "initial_release_date": "1979-05-25T00:00:00Z"
          },
          {
            "name@en": "Black Rain",
            "initial_release_date": "1989-09-22T00:00:00Z"
          },
          {
            "name@en": "White Squall",
            "initial_release_date": "1996-02-02T00:00:00Z"
          },
          {
            "name@en": "Thelma & Louise",
            "initial_release_date": "1991-05-24T00:00:00Z"
          },
          {
            "name@en": "G.I. Jane",
            "initial_release_date": "1997-08-22T00:00:00Z"
          },
          {
            "name@en": "1492 Conquest of Paradise",
            "initial_release_date": "1992-10-08T00:00:00Z"
          },
          {
            "name@en": "The Duellists",
            "initial_release_date": "1977-12-01T00:00:00Z"
          },
          {
            "name@en": "Someone to Watch Over Me",
            "initial_release_date": "1987-10-09T00:00:00Z"
          },
          {
            "name@en": "Legend",
            "initial_release_date": "1985-08-28T00:00:00Z"
          },
          {
            "name@en": "Boy and Bicycle",
            "initial_release_date": "1997-09-07T00:00:00Z"
          }
        ]
      },
      {
        "name@en": "Ridley Scott"
      }
    ]
  }
}
```

查询示例：2000年之前上映的片名为`Blade`或`Runner`的电影：

``` dql
{
  bladerunner(func: anyofterms(name@en, "Blade Runner")) @filter(le(initial_release_date, "2000")) {
    uid
    name@en
    initial_release_date
    netflix_id
  }
}
```

将返回：
``` json
{
  "data": {
    "bladerunner": [
      {
        "uid": "0x394c",
        "name@en": "Blade Runner",
        "initial_release_date": "1982-06-25T00:00:00Z",
        "netflix_id": "70083726"
      },
      {
        "uid": "0x7160",
        "name@en": "Blade",
        "initial_release_date": "1973-12-01T00:00:00Z"
      },
      {
        "uid": "0x15adb",
        "name@en": "Lone Runner",
        "initial_release_date": "1986-01-01T00:00:00Z",
        "netflix_id": "70146965"
      },
      {
        "uid": "0x16f0c",
        "name@en": "Revenge of the Bushido Blade",
        "initial_release_date": "1980-01-01T00:00:00Z",
        "netflix_id": "70100967"
      }
      ...
    ]
  }
}
```

## 多语言支持
> 注意：必须在`Schema`中指定`@lang`指令来查询或修改带有语言标记的`predicate`。

`Dgraph`支持`UTF-8`

使用以下规则指定返回语言的优先顺序：
* 最多只会返回一个结果（除了语言列表被设置为*的情况）。
* 从左到右考虑首选项列表：如果没有找到给定语言的值，则考虑列表中的下一种语言。
* 如果在任何指定的语言中没有值，则不返回值。
* 最后一个`.`表示返回没有指定语言的值，或者如果没有没有语言的值，则返回`some`语言的值。
* 将语言列表值设置为`*`将返回该谓词的所有值及其语言。也会返回没有语言标记的值。
例如：
* `name`: 查找一个`untagged`的字符串;如果没有`untagged`的值存在，则不返回任何值。
* `name@.`: 查找一个未标记的字符串，然后是任何语言。
* `name@en`: 查找`zh`标记的字符串；如果不存在带`en`标记的字符串，则不返回任何内容。
* `name@en:.`: 查找`en`，然后`untagged`，然后是任何语言。
* `name@en:pl`: 查找`en`，然后是`pl`，否则什么都不返回。
* `name@*`: 查找该谓词的所有值，并返回它们及其语言。例如，如果有两个语言为`en`和`hi`的值，则该查询将返回两个名为`name@en`和`name@hi`的键。

> 注意，在函数中，语言列表（包括`@*`符号）是不允许的。无标记谓词、单语言标记和`.`符号如上所述。
> 在全文搜索函数（`alloftext`、`anyoftext`）中，当没有指定语言（`untagged`或`@.`）时，将使用默认的（英文）全文标记器。这并不意味着在查询无标记值时将搜索带有en标记的值，而是将无标记值视为英语文本。如果您不希望出现这种情况，可以为所需的语言使用适当的标记，用于修改和查询值。

查询示例:宝莱坞导演和演员法尔汉·阿赫塔尔（Farhan Akhtar）的一些电影有俄语、印地语和英语的名字，其他的没有：

``` dql
{
  q(func: allofterms(name@en, "Farhan Akhtar")) {
    name@.
    director.film {
      name@ru:hi:en
      name@en
      name@hi
      name@ru
    }
  }
}
```

将返回：

``` json 
{
  "data": {
    "q": [
      {
        "name@.": "Farhan Akhtar",
        "director.film": [
          {
            "name@ru:hi:en": "दिल चाहता है",
            "name@en": "Dil Chahta Hai",
            "name@hi": "दिल चाहता है"
          },
          {
            "name@ru:hi:en": "Дон. Главарь мафии 2",
            "name@en": "Don 2",
            "name@hi": "डॉन २",
            "name@ru": "Дон. Главарь мафии 2"
          },
          {
            "name@ru:hi:en": "पोज़िटिव",
            "name@en": "Positive",
            "name@hi": "पोज़िटिव"
          },
          {
            "name@ru:hi:en": "लक्ष्य",
            "name@en": "Lakshya",
            "name@hi": "लक्ष्य"
          },
          {
            "name@ru:hi:en": "Дон. Главарь мафии",
            "name@en": "Don: The Chase Begins Again",
            "name@hi": "डॉन",
            "name@ru": "Дон. Главарь мафии"
          }
        ]
      }
    ]
  }
}
```