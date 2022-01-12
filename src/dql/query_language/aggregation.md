# 聚合操作 Aggregation

解析规则：
* `AG(val(varName))`

`AG`的可能取值是：
* `min`: 在值变量`varName`里选择最小值
* `max`: 在值变量`varName`里选择最大值
* `sum`: 计算总和
* `avg`: 计算`varName`的平均数

Schema类型：

操作 | Schema类型
----| ----
min/max|int float string dateTime default
sum/avg|int float

聚合操作只能用在值变量上，不需要索引。

在包含变量定义的查询块上应用聚合。与查询变量和值变量(它们是全局的)不同，聚合是在本地计算的。例如:

``` graphql
A as predicateA {
  ...
  B as predicateB {
    x as ...some value...
  }
  min(val(x))
}
```

## Min

### 用在根查询中

### 其他用法

查询示例：`Steven`导演的，并按第一部电影的上映日期升序排列的电影：

``` graphql
{
  stevens as var(func: allofterms(name@en, "steven")) {
    director.film {
      ird as initial_release_date
      # ird is a value variable mapping a film UID to its release date
    }
    minIRD as min(val(ird))
    # minIRD is a value variable mapping a director UID to their first release date
  }

  byIRD(func: uid(stevens), orderasc: val(minIRD)) {
    name@en
    firstRelease: val(minIRD)
  }
}
```

``` json 
{
  "data": {
    "byIRD": [
      {
        "name@en": "Steven McMillan",
        "firstRelease": "0214-02-28T00:00:00Z"
      },
      {
        "name@en": "J. Steven Edwards",
        "firstRelease": "1929-01-01T00:00:00Z"
      },
      {
        "name@en": "Steven Spielberg",
        "firstRelease": "1964-03-24T00:00:00Z"
      }
      ...
    ]
  }
}
```

## Max

### 用在根查询中

查询示例：获取名字包含`Harry Potter`的最新更新的电影，
更新日期被映射到一个变量(d)，然后被聚合操作(max)，最后附加到一个空的查询块中：

``` graphql
{
  var(func: allofterms(name@en, "Harry Potter")) {
    d as initial_release_date
  }
  me() {
    max(val(d))
  }
}
```

``` json 
{
  "data": {
    "me": [
      {
        "max(val(d))": "2011-07-27T00:00:00Z"
      }
    ]
  }
}
```


### 其它用法

查询示例：`Quentin Tarantino`发布的电影，按时间最新的排列：

``` graphql
{
  director(func: allofterms(name@en, "Quentin Tarantino")) {
    director.film {
      name@en
      x as initial_release_date
    }
    max(val(x))
  }
}
```


``` json 
{
  "data": {
    "director": [
      {
        "director.film": [
          {
            "name@en": "Kill Bill Volume 1",
            "initial_release_date": "2003-10-10T00:00:00Z"
          },
          {
            "name@en": "Django Unchained",
            "initial_release_date": "2012-12-25T00:00:00Z"
          },
          {
            "name@en": "Sin City",
            "initial_release_date": "2005-03-28T00:00:00Z"
          }
          ...
        ]
      }
    ]
  }
}
```

## 总和和平均数 `Sum`, `Avg`

### 用在根查询中

### 其它用法


## Aggregating Aggregates

可以将聚合分配给值变量，因此可以依次对这些变量进行聚合。

查询示例：对于彼得·杰克逊电影中的每个演员，找出在任何电影中扮演的角色的数量。把这些数字加起来，就能算出所有演员在这部电影中所扮演的角色总数。然后把这些数字加起来，就能算出演员们在彼得·杰克逊的电影中扮演过的角色总数。注意，这演示了如何聚合聚合；不过，这个问题的答案并不十分精确，因为在彼得·杰克逊的多部电影中出现的演员被计算了不止一次。

``` graphql
{
  PJ as var(func:allofterms(name@en, "Peter Jackson")) {
    director.film {
      starring {  # starring an actor
        performance.actor {
          movies as count(actor.film)
          # number of roles for this actor
        }
        perf_total as sum(val(movies))
      }
      movie_total as sum(val(perf_total))
      # total roles for all actors in this movie
    }
    gt as sum(val(movie_total))
  }

  PJmovies(func: uid(PJ)) {
    name@en
    director.film (orderdesc: val(movie_total), first: 5) {
      name@en
      totalRoles : val(movie_total)
    }
    grandTotal : val(gt)
  }
}
```

``` json 
{
  "data": {
    "PJmovies": [
      {
        "name@en": "Peter Jackson",
        "director.film": [
          {
            "name@en": "The Lord of the Rings: The Two Towers",
            "totalRoles": 1565
          },
          {
            "name@en": "The Lord of the Rings: The Return of the King",
            "totalRoles": 1518
          },
          {
            "name@en": "The Lord of the Rings: The Fellowship of the Ring",
            "totalRoles": 1335
          },
          {
            "name@en": "The Hobbit: An Unexpected Journey",
            "totalRoles": 1274
          },
          {
            "name@en": "The Hobbit: The Desolation of Smaug",
            "totalRoles": 1261
          }
        ],
        "grandTotal": 10592
      },
      {
        "name@en": "Peter Jackson"
      },
      {
        "name@en": "Peter Jackson"
      },
      {
        "name@en": "Peter Jackson"
      },
      {
        "name@en": "Sam Peter Jackson"
      }
    ]
  }
}
```