# facet和Edge属性

`Dgraph`支持`facet`————`edge`上的键值对————是`RDF`三元组的扩展。也就是说，`facet`向边添加属性，而不是向节点添加属性。例如，两个节点之间的朋友边可能具有亲密友谊的`bool`属性。`facet`也可以用作边（edge）的权重。

虽然你可能会发现自己很多时候倾向于一些方面，但它们不应该被滥用。例如，给`friend`这条边（edge）添加一个`date_of_birth`属性可能不是一个正经的建模。对于`friend`这条边，你更应该添加比如`start_of_friendship`这个属性。然而，`facet`在`Dgraph`中并不像谓词（`predicate`）那样是一等公民。

`Facet`键是字符串，值可以是`string`、`bool`、`int`、`float`和`dateTime`。对于`int`和`float`，只接受32位有符号整数和64位浮点数。

下面这些对于`facet`的变更将会贯穿整章。该变更（`mutation`）为一些人添加了数据，例如，在`mobile`和`car`中记录了`since` `facet`，以记录`Alice`购买汽车并开始使用手机号码的时间。

首先，我们添加一些`Schema`：

``` bash
curl localhost:8080/alter -XPOST -d $'
    name: string @index(exact, term) .
    rated: [uid] @reverse @count .
' | python -m json.tool | less
```

``` bash
curl -H "Content-Type: application/rdf" localhost:8080/mutate?commitNow=true -XPOST -d $'
{
  set {

    # -- Facets on scalar predicates
    _:alice <name> "Alice" .
    _:alice <dgraph.type> "Person" .
    _:alice <mobile> "040123456" (since=2006-01-02T15:04:05) .
    _:alice <car> "MA0123" (since=2006-02-02T13:01:09, first=true) .

    _:bob <name> "Bob" .
    _:bob <dgraph.type> "Person" .
    _:bob <car> "MA0134" (since=2006-02-02T13:01:09) .

    _:charlie <name> "Charlie" .
    _:charlie <dgraph.type> "Person" .
    _:dave <name> "Dave" .
    _:dave <dgraph.type> "Person" .


    # -- Facets on UID predicates
    _:alice <friend> _:bob (close=true, relative=false) .
    _:alice <friend> _:charlie (close=false, relative=true) .
    _:alice <friend> _:dave (close=true, relative=true) .


    # -- Facets for variable propagation
    _:movie1 <name> "Movie 1" .
    _:movie1 <dgraph.type> "Movie" .
    _:movie2 <name> "Movie 2" .
    _:movie2 <dgraph.type> "Movie" .
    _:movie3 <name> "Movie 3" .
    _:movie3 <dgraph.type> "Movie" .

    _:alice <rated> _:movie1 (rating=3) .
    _:alice <rated> _:movie2 (rating=2) .
    _:alice <rated> _:movie3 (rating=5) .

    _:bob <rated> _:movie1 (rating=5) .
    _:bob <rated> _:movie2 (rating=5) .
    _:bob <rated> _:movie3 (rating=5) .

    _:charlie <rated> _:movie1 (rating=2) .
    _:charlie <rated> _:movie2 (rating=5) .
    _:charlie <rated> _:movie3 (rating=1) .
  }
}' | python -m json.tool | less
```

## 标量(scalar)谓词(predicate)上的Facets

查询`Alice`的`name`、`mobile`和`Car`与不查询`facet`的结果相同：

``` dql
{
  data(func: eq(name, "Alice")) {
    name
    mobile
    car
  }
}
```

``` json 
{
  "data": {
    "data": [
      {
        "name": "Alice",
        "mobile": "040123456",
        "car": "MA0123"
      }
    ]
  }
}
```

语法`@facet (facet-name)`用于查询`facet`数据。对于`Alice`, `mobile`和`car`的`since`这个`facet`被查询如下：

``` dql
{
  data(func: eq(name, "Alice")) {
    name
    mobile @facets(since)
    car @facets(since)
  }
}
```

``` json
{
  "data": {
    "data": [
      {
        "name": "Alice",
        "mobile|since": "2006-01-02T15:04:05Z",
        "mobile": "040123456",
        "car|since": "2006-02-02T13:01:09Z",
        "car": "MA0123"
      }
    ]
  }
}
```

`facet`在与对应的边(edge)相同的级别返回，并具有像`edge|facet`这样的键。

边(edge)上的所有`facet`都使用`@facet`查询：

``` dql 
{
  data(func: eq(name, "Alice")) {
    name
    mobile @facets
    car @facets
  }
}
```

``` json 
{
  "data": {
    "data": [
      {
        "name": "Alice",
        "mobile|since": "2006-01-02T15:04:05Z",
        "mobile": "040123456",
        "car|first": true,
        "car|since": "2006-02-02T13:01:09Z",
        "car": "MA0123"
      }
    ]
  }
}
```

## Facets 国际化

`facet`键和值在发生变化时可以直接使用特定于语言的字符。但是在查询时，`facet`键需要括在尖括号<>中。这类似于谓词。有关更多信息，请参阅谓词(predicate)[i18n](https://dgraph.io/docs/query-language/schema/#predicates-i18n)。

> 注意，在查询时，Dgraph支持面向键的国际化资源标识符(IRIs)。

``` dql 
{
  set {
    _:person1 <name> "Daniel" (वंश="स्पेनी", ancestry="Español") .
    _:person1 <dgraph.type> "Person" .
    _:person2 <name> "Raj" (वंश="हिंदी", ancestry="हिंदी") .
    _:person2 <dgraph.type> "Person" .
    _:person3 <name> "Zhang Wei" (वंश="चीनी", ancestry="中文") .
    _:person3 <dgraph.type> "Person" .
  }
}
```

查询，注意`<>`：

``` dql
{
  q(func: has(name)) {
    name @facets(<वंश>)
  }
}
```

## 在Facets上使用别名

可以在请求特定谓词(predicate)时指定别名。语法类似于为其他谓词请求别名的方式。`orderasc`和`orderdesc`不允许作为别名，因为它们有特殊的含义。除此之外，其他任何东西都可以设置为别名。

这里我们分别为`since`、`close` facet设置`car_since`、`close_friend`别名：

``` dql
{
  data(func: eq(name, "Alice")) {
  name
  mobile
  car @facets(car_since: since)
  friend @facets(close_friend: close) {
    name
  }
  }
}
```

``` dql
{
  "data": {
    "data": [
      {
        "name": "Alice",
        "mobile": "040123456",
        "car_since": "2006-02-02T13:01:09Z",
        "car": "MA0123",
        "friend": [
          {
            "name": "Bob",
            "close_friend": true
          },
          {
            "name": "Charlie",
            "close_friend": false
          },
          {
            "name": "Dave",
            "close_friend": true
          }
        ]
      }
    ]
  }
}
```

## `UID`谓词(predicate)上的Facets

`UID`边(edge)上的`facet`与值边(value edge)上的`facet`的工作方式类似。

例如，`friend`这条边拥有一个`close`的`edge`。`Alice`和`Bob`的`friend`边(edge)的`close``facet`被设置为true，`Alice`和`Charlie`之间被设置为`false`。

现在查询`Alice`的朋友：

``` dql
{
  data(func: eq(name, "Alice")) {
    name
    friend {
      name
    }
  }
}
```

``` json
{
  "data": {
    "data": [
      {
        "name": "Alice",
        "friend": [
          {
            "name": "Bob"
          },
          {
            "name": "Charlie"
          },
          {
            "name": "Dave"
          }
        ]
      }
    ]
  }
}
```

查询`Alice`的朋友这条边(edge)上的`close``facet`为`false`的朋友：

``` dql
{
  data(func: eq(name, "Alice")) {
    name
    friend @facets(close) {
      name
    }
  }
}
```

``` json
{
  "data": {
    "data": [
      {
        "name": "Alice",
        "friend": [
          {
            "name": "Bob",
            "friend|close": true
          },
          {
            "name": "Charlie",
            "friend|close": false
          },
          {
            "name": "Dave",
            "friend|close": true
          }
        ]
      }
    ]
  }
}
```

对于像`friend`这样的`UID`边(edge)，`facet`会转到相应子元素下。在上面的示例中，您可以看到`Alice`和`Bob`之间的边(edge)上的`close``facet`。

``` dql
{
  data(func: eq(name, "Alice")) {
    name
    friend @facets {
      name
      car @facets
    }
  }
}
```

``` json 
{
  "data": {
    "data": [
      {
        "name": "Alice",
        "friend": [
          {
            "name": "Bob",
            "car|since": "2006-02-02T13:01:09Z",
            "car": "MA0134",
            "friend|close": true,
            "friend|relative": false
          },
          {
            "name": "Charlie",
            "friend|close": false,
            "friend|relative": true
          },
          {
            "name": "Dave",
            "friend|close": true,
            "friend|relative": true
          }
        ]
      }
    ]
  }
}
```

`Bob`有一辆车，它有一个`facet` `since`，在结果中，它是`key car|since`下`Bob`的同一个对象的一部分。此外，`Bob`和`Alice`之间的密切关系也是`Bob`的输出对象的一部分。`Charlie`没有`Car`边（edge），因此只有`UID` `facets`。

## 在facets上使用过滤

`Dgraph`支持基于`facet`的边（edge）过滤。过滤的工作原理类似于它在没有`facet`的边（edge）上的工作原理，并且具有相同的可用函数。

找到爱丽丝的好朋友：

``` dql
{
  data(func: eq(name, "Alice")) {
    friend @facets(eq(close, true)) {
      name
    }
  }
}
```


``` json 
{
  "data": {
    "data": [
      {
        "friend": [
          {
            "name": "Bob"
          },
          {
            "name": "Dave"
          }
        ]
      }
    ]
  }
}

```

要返回facet和过滤器，请向查询添加另一个`@facet (<facetname>)`：

``` dql
{
  data(func: eq(name, "Alice")) {
    friend @facets(eq(close, true)) @facets(relative) { # filter close friends and give relative status
      name
    }
  }
}
```

``` json 
{
  "data": {
    "data": [
      {
        "friend": [
          {
            "name": "Bob",
            "friend|relative": false
          },
          {
            "name": "Dave",
            "friend|relative": true
          }
        ]
      }
    ]
  }
}
```

`Facet`查询能够和`AND`, `OR`, `NOT` 组合使用：

``` dql
{
  data(func: eq(name, "Alice")) {
    friend @facets(eq(close, true) AND eq(relative, true)) @facets(relative) { # filter close friends in my relation
      name
    }
  }
}
```

``` json 
{
  "data": {
    "data": [
      {
        "friend": [
          {
            "name": "Dave",
            "friend|relative": true
          }
        ]
      }
    ]
  }
}
```


## 使用`Facet`排序

可以对`uid`边（edge）上的`facet`进行排序。在这里，我们将爱丽丝、鲍勃和查理对电影的评级进行分类，这是一个`facet`：

``` dql
{
  me(func: anyofterms(name, "Alice Bob Charlie")) {
    name
    rated @facets(orderdesc: rating) {
      name
    }
  }
}
```

``` json 
{
  "data": {
    "me": [
      {
        "name": "Bob",
        "rated": [
          {
            "name": "Movie 1",
            "rated|rating": 5
          },
          {
            "name": "Movie 2",
            "rated|rating": 5
          },
          {
            "name": "Movie 3",
            "rated|rating": 5
          }
        ]
      },
      {
        "name": "Alice",
        "rated": [
          {
            "name": "Movie 3",
            "rated|rating": 5
          },
          {
            "name": "Movie 1",
            "rated|rating": 3
          },
          {
            "name": "Movie 2",
            "rated|rating": 2
          }
        ]
      },
      {
        "name": "Charlie",
        "rated": [
          {
            "name": "Movie 2",
            "rated|rating": 5
          },
          {
            "name": "Movie 1",
            "rated|rating": 2
          },
          {
            "name": "Movie 3",
            "rated|rating": 1
          }
        ]
      }
    ]
  }
}
```

## 将`Facet`值赋值给变量

`UID`边（edge）上的`facet`可以存储在值变量（variables）中。这个变量是从边（edge）目标到`facet`的映射。

找到`Alice`被标记为`close`和`relative`的朋友：

``` dql
{
  var(func: eq(name, "Alice")) {
    friend @facets(a as close, b as relative)
  }

  friend(func: uid(a)) {
    name
    val(a)
  }

  relative(func: uid(b)) {
    name
    val(b)
  }
}
```

``` json 
{
  "data": {
    "friend": [
      {
        "name": "Bob",
        "val(a)": true
      },
      {
        "name": "Charlie",
        "val(a)": false
      },
      {
        "name": "Dave",
        "val(a)": true
      }
    ],
    "relative": [
      {
        "name": "Bob",
        "val(b)": false
      },
      {
        "name": "Charlie",
        "val(b)": true
      },
      {
        "name": "Dave",
        "val(b)": true
      }
    ]
  }
}
```

## Facet和值传播

可以将`int`和`float`的`Facet`值赋给变量，从而使这些值（variables）传播。

`Alice`、`Bob`和`Charlie`都给每部电影打分。`facet`评级上的值变量将电影映射到评级。通过多条路径到达电影的查询将对每条路径的评级进行求和。下面总结了`Alice`、`Bob`和`Charlie`对这三部电影的评分：

``` dql
{
  var(func: anyofterms(name, "Alice Bob Charlie")) {
    num_raters as math(1)
    rated @facets(r as rating) {
      total_rating as math(r) # sum of the 3 ratings
      average_rating as math(total_rating / num_raters)
    }
  }
  data(func: uid(total_rating)) {
    name
    val(total_rating)
    val(average_rating)
  }

}
```

``` json 
{
  "data": {
    "data": [
      {
        "name": "Movie 1",
        "val(total_rating)": 10,
        "val(average_rating)": 3
      },
      {
        "name": "Movie 2",
        "val(total_rating)": 12,
        "val(average_rating)": 4
      },
      {
        "name": "Movie 3",
        "val(total_rating)": 11,
        "val(average_rating)": 3
      }
    ]
  }
}
```

## Facet 和聚合(aggregation)

可以聚合分配给值变量的Facet值。

``` dql
{
  data(func: eq(name, "Alice")) {
    name
    rated @facets(r as rating) {
      name
    }
    avg(val(r))
  }
}
```

``` json 
{
  "data": {
    "data": [
      {
        "name": "Alice",
        "rated": [
          {
            "name": "Movie 1",
            "rated|rating": 3
          },
          {
            "name": "Movie 2",
            "rated|rating": 2
          },
          {
            "name": "Movie 3",
            "rated|rating": 5
          }
        ],
        "avg(val(r))": 3.333333
      }
    ]
  }
}
```

但是请注意，`r`是一个从电影到查询中到达电影的边缘上的评分总和的映射。因此，下面的代码不能正确地分别计算`Alice`和`Bob`的平均评级————它计算的是`Alice`和`Bob`评级平均值的2倍。

``` dql
{
  data(func: anyofterms(name, "Alice Bob")) {
    name
    rated @facets(r as rating) {
      name
    }
    avg(val(r))
  }
}
```

``` json 
{
  "data": {
    "data": [
      {
        "name": "Bob",
        "rated": [
          {
            "name": "Movie 1",
            "rated|rating": 5
          },
          {
            "name": "Movie 2",
            "rated|rating": 5
          },
          {
            "name": "Movie 3",
            "rated|rating": 5
          }
        ],
        "avg(val(r))": 8.333333
      },
      {
        "name": "Alice",
        "rated": [
          {
            "name": "Movie 1",
            "rated|rating": 3
          },
          {
            "name": "Movie 2",
            "rated|rating": 2
          },
          {
            "name": "Movie 3",
            "rated|rating": 5
          }
        ],
        "avg(val(r))": 8.333333
      }
    ]
  }
}
```

计算用户的平均评级需要一个将用户映射到其评级总和的变量：

``` dql
{
  var(func: has(rated)) {
    num_rated as math(1)
    rated @facets(r as rating) {
      avg_rating as math(r / num_rated)
    }
  }

  data(func: uid(avg_rating)) {
    name
    val(avg_rating)
  }
}

```


``` json 
{
  "data": {
    "data": [
      {
        "name": "Movie 1",
        "val(avg_rating)": 3
      },
      {
        "name": "Movie 2",
        "val(avg_rating)": 4
      },
      {
        "name": "Movie 3",
        "val(avg_rating)": 3
      }
    ]
  }
}
```

