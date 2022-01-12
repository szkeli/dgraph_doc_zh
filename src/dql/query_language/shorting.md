# Shorting 排序

语法示例：
* `q(func: ..., orderasc: predicate)`
* `q(func: ..., orderdesc: val(name))`
* `predicate (orderdesc: predicate) { ... }`
* `predicate @filter(...) (orderasc: N) { ... }`
* `q(func: ..., orderasc: predicate1, orderdesc: predicate2)`

可排序类型：`int`, `float`, `String`, `dateTime`, `default`

结果可以按谓词(predicate)或变量(variable)的升序(`orderasc`)或降序(`orderdesc`)排序。

对于具有[可排序索引](./schema.md#sortable-indices)的谓词的排序，`Dgraph`并行地对值(value)和索引(index)进行排序，并返回先计算完的结果。

> 注意，`Dgraph`在结果的末尾返回`null`，而不考虑其类型。这种行为在索引排序和非索引排序中是一致的。

> 默认情况下，排序查询最多可检索1000个结果。这可以用`first`更改。

查询示例：法国导演`Jean-Pierre Jeunet`按上映日期排序的电影：

``` dql
{
  me(func: allofterms(name@en, "Jean-Pierre Jeunet")) {
    name@fr
    director.film(orderasc: initial_release_date) {
      name@fr
      name@en
      initial_release_date
    }
  }
}
```

``` json 
{
  "data": {
    "me": [
      {
        "name@fr": "Jean-Pierre Jeunet",
        "director.film": [
          {
            "name@fr": "L'Évasion",
            "name@en": "L'évasion",
            "initial_release_date": "1978-01-01T00:00:00Z"
          },
          {
            "name@fr": "Le Manège",
            "name@en": "Le manège",
            "initial_release_date": "1980-01-01T00:00:00Z"
          }
          ...
        ]
      }
    ]
  }
}
```

排序可以在根查询或值变量上进行。

查询示例：所有流派按字母表排序，每个类型中类型最多的5部电影：

``` dql
{
  genres as var(func: has(~genre)) {
    ~genre {
      numGenres as count(genre)
    }
  }

  genres(func: uid(genres), orderasc: name@en) {
    name@en
    ~genre (orderdesc: val(numGenres), first: 5) {
      name@en
      genres : val(numGenres)
    }
  }
}
```

``` json
{
  "data": {
    "genres": [
      {
        "name@en": "/m/04rlf",
        "~genre": [
          {
            "name@en": "Bookin'",
            "genres": 3
          },
          {
            "name@en": "La brune et moi",
            "genres": 3
          }
        ]
      },
      {
        "name@en": "3D film",
        "~genre": [
          {
            "name@en": "Mr. X",
            "genres": 4
          },
          {
            "name@en": "Sly Cooper",
            "genres": 4
          },
          {
            "name@en": "Everest",
            "genres": 3
          },
          {
            "name@en": "Aqaye Alef",
            "genres": 1
          }
        ]
      },
      {
        "name@en": "Abstract animation",
        "~genre": [
          {
            "name@en": "The Heart of the World",
            "genres": 7
          },
          {
            "name@en": "Outside Out",
            "genres": 6
          },
          {
            "name@en": "11 x 14",
            "genres": 5
          },
          {
            "name@en": "Acousticity",
            "genres": 4
          },
          {
            "name@en": "The Garden of Earthly Delights",
            "genres": 4
          }
        ]
      }
      ...
    ]
  }
}
```

排序也可以同时对多个谓语进行，如下所示。如果第一个谓词的值相等，则按第二个谓词对它们进行排序，以此类推。

查询示例：查找所有具有`Person`类型的节点，按`first_name`对其进行排序，`first_name`相同的节点按`last_name`降序进行排序。

``` dql
{
  me(func: type("Person"), orderasc: first_name, orderdesc: last_name) {
    first_name
    last_name
  }
}
```