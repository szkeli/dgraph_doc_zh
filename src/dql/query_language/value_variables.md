# 值变量

解析示例：
* `varName as scalarPredicate`
* `varName as count(predicate)`
* `varName as avg(...)`
* `varName as math(...)`

支持的类型：`int`, `float`, `String`, `dateTime`, `geo`, `default`, `bool`

值变量存储标量值。值变量是从封闭块的`uid`到相应值的映射。

因此，只有在匹配相同`uid`的上下文中使用值变量的值才有意义————如果在匹配不同`uid`的块中使用值变量是未定义的。

定义值变量而不在查询的其他地方使用它是错误的。

值变量是通过使用`val(var-name)`提取值，或使用`uid(var-name)`提取`uid`来使用的。

[Facet](https://dgraph.io/docs/query-language/facets/)值可以存储在值变量中。

查询示例：80年代经典电影《公主新娘》中，演员所扮演的电影角色数量。查询变量pbActors匹配电影中所有演员的uid。因此，值变量角色是一个从actor UID到角色数量的映射。值变量角色可以在totalRoles查询块中使用，因为该查询块也匹配pbActors uid，因此actor到角色数量的映射是可用的：

``` graphql
{
  var(func:allofterms(name@en, "The Princess Bride")) {
    starring {
      pbActors as performance.actor {
        roles as count(actor.film)
      }
    }
  }
  totalRoles(func: uid(pbActors), orderasc: val(roles)) {
    name@en
    numRoles : val(roles)
  }
}
```

``` json
{
  "data": {
    "totalRoles": [
      {
        "name@en": "Mark Knopfler",
        "numRoles": 1
      },
      {
        "name@en": "Annie Dyson",
        "numRoles": 2
      },
      {
        "name@en": "André the Giant",
        "numRoles": 7
      }
      ...
    ]
  }
}
```

值变量可以通过从映射中提取`UID`列表来代替`UID`变量。

查询示例：与前面的示例相同的查询，但是使用值变量角色来匹配`totalRoles`查询块中的`uid`：

``` graphql
{
  var(func:allofterms(name@en, "The Princess Bride")) {
    starring {
      performance.actor {
        roles as count(actor.film)
      }
    }
  }
  totalRoles(func: uid(roles), orderasc: val(roles)) {
    name@en
    numRoles : val(roles)
  }
}
```

``` json 
{
  "data": {
    "totalRoles": [
      {
        "name@en": "Mark Knopfler",
        "numRoles": 1
      },
      {
        "name@en": "Annie Dyson",
        "numRoles": 2
      },
      {
        "name@en": "André the Giant",
        "numRoles": 7
      }
      ...
    ]
  }
}
```

## 变量传播

与查询变量一样，值变量可以在其他查询块中使用，也可以在定义块内嵌套的块中使用。当在定义变量的块内嵌套的块中使用时，该值作为父节点到使用点的所有路径上的变量的和计算。这叫做变量传播：

``` graphql
{
  q(func: uid(0x01)) {
    myscore as math(1)          # A
    friends {                   # B
      friends {                 # C
        ...myscore...
      }
    }
  }
}
```

在第A行，一个值变量`myscore`被定义为`UID`为`0x01`的节点映射到值`1`。在B处，每个朋友的值仍然是`1`:每个朋友只有一条路径。穿过朋友的边缘两次，就会到达朋友的朋友。变量`myscore`将被传播，这样每个朋友的朋友将收到它的父母值的总和:如果一个朋友的朋友只能从一个朋友处到达，那么这个值仍然是1，如果他们可以从两个朋友处到达，那么这个值是2，以此类推。也就是说，标记为C的块中每个朋友的朋友的`myscore`值将是到他们的路径数。

**节点接收到的传播变量的值是其所有父节点值的和。**

这种传播是有用的，例如，在标准化用户之间的和、查找节点之间的路径数量和通过图积累一个和时。

查询示例：每一部《哈利·波特》电影中，沃里克·戴维斯扮演的角色数量：

``` graphql
{
    num_roles(func: eq(name@en, "Warwick Davis")) @cascade @normalize {

    paths as math(1)  # records number of paths to each character

    actor : name@en

    actor.film {
      performance.film @filter(allofterms(name@en, "Harry Potter")) {
        film_name : name@en
        characters : math(paths)  # how many paths (i.e. characters) reach this film
      }
    }
  }
}
```

``` json 
{
  "data": {
    "num_roles": [
      {
        "actor": "Warwick Davis",
        "film_name": "Harry Potter and the Prisoner of Azkaban",
        "characters": 1
      },
      {
        "actor": "Warwick Davis",
        "film_name": "Harry Potter and the Goblet of Fire",
        "characters": 1
      },
      {
        "actor": "Warwick Davis",
        "film_name": "Harry Potter and the Deathly Hallows – Part 2",
        "characters": 2
      },
      {
        "actor": "Warwick Davis",
        "film_name": "Harry Potter and the Deathly Hallows - Part I",
        "characters": 1
      },
      {
        "actor": "Warwick Davis",
        "film_name": "Harry Potter and the Order of the Phoenix",
        "characters": 1
      },
      {
        "actor": "Warwick Davis",
        "film_name": "Harry Potter and the Philosopher's Stone",
        "characters": 2
      },
      {
        "actor": "Warwick Davis",
        "film_name": "Harry Potter and the Philosopher's Stone",
        "characters": 2
      },
      {
        "actor": "Warwick Davis",
        "film_name": "Harry Potter and the Half-Blood Prince",
        "characters": 1
      },
      {
        "actor": "Warwick Davis",
        "film_name": "Harry Potter and the Deathly Hallows – Part 2",
        "characters": 2
      },
      {
        "actor": "Warwick Davis",
        "film_name": "Harry Potter and the Chamber of Secrets",
        "characters": 1
      }
    ]
  }
}
```

查询示例：在彼得·杰克逊的电影中出演过的每个演员以及他们在彼得·杰克逊的电影中出演的比例：

``` graphql
{
    movie_fraction(func:eq(name@en, "Peter Jackson")) @normalize {

    paths as math(1)
    total_films : num_films as count(director.film)
    director : name@en

    director.film {
      starring {
        performance.actor {
          fraction : math(paths / (num_films/paths))
          actor : name@en
        }
      }
    }
  }
}
```

``` json
{
  "data": {
    "movie_fraction": [
      {
        "total_films": 19,
        "director": "Peter Jackson",
        "fraction": 0,
        "actor": "Pete O'Herne"
      },
      {
        "total_films": 19,
        "director": "Peter Jackson",
        "fraction": 0,
        "actor": "Peter Jackson"
      },
      {
        "total_films": 19,
        "director": "Peter Jackson",
        "fraction": 0,
        "actor": "Peter Jackson"
      }
      ...
    ]
  }
}
```

更多的例子可以在两篇`Dgraph`博客文章中找到([post1](https://open.dgraph.io/post/recommendation/), [post2](https://open.dgraph.io/post/recommendation2/))，这两篇文章是关于为推荐引擎使用变量传播的。