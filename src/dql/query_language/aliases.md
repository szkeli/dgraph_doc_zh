# Aliases 别名

解析示例：
* aliasName: predicate
* aliasName: predicate { ... }
* aliasName: varName as ...
* aliasName: count(predicate)
* aliasName: max(val(varName))

别名在结果中提供替代名称。谓词、变量和聚合可以通过以别名和:作为前缀来别名。别名不必与原始谓词名不同，但是，在一个块中，别名必须与谓词名和同一块中返回的其他别名不同。别名可用于在一个块内多次返回相同的谓词。

查询示例：姓名匹配词`Steven`的导演，他们的`UID`，英文名，每部电影的平均演员人数，电影总数，以及每部电影的英文和法文名称：

``` graphql
{
  ID as var(func: allofterms(name@en, "Steven")) @filter(has(director.film)) {
    director.film {
      num_actors as count(starring)
    }
    average as avg(val(num_actors))
  }

  films(func: uid(ID)) {
    director_id : uid
    english_name : name@en
    average_actors : val(average)
    num_films : count(director.film)

    films : director.film {
      name : name@en
      english_name : name@en
      french_name : name@fr
    }
  }
}
```

将返回：

``` json
{
  "data": {
    "films": [
      {
        "director_id": "0x33a6",
        "english_name": "Steven Wilsey",
        "average_actors": 0,
        "num_films": 1,
        "films": [
          {
            "name": "At Night I Was Beautiful",
            "english_name": "At Night I Was Beautiful"
          }
        ]
      },
      {
        "director_id": "0x3ce3",
        "english_name": "Steven Bratter",
        "average_actors": 10,
        "num_films": 1,
        "films": [
          {
            "name": "First Strike",
            "english_name": "First Strike"
          }
        ]
      },
      {
        "director_id": "0x57c8",
        "english_name": "Steven Saussey",
        "average_actors": 3,
        "num_films": 1,
        "films": [
          {
            "name": "Whisker",
            "english_name": "Whisker"
          }
        ]
      },
      {
        "director_id": "0x9740",
        "english_name": "Steven Ray Morris",
        "average_actors": 8,
        "num_films": 1,
        "films": [
          {
            "name": "The Premiere",
            "english_name": "The Premiere"
          }
        ]
      }
      ...
    ]
  }
}
```

