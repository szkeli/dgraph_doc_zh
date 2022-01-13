# 查询变量

语法解析：

* `varName as q(func: ...) { ... }`
* `varName as var(func: ...) { ... }`
* `varName as predicate { ... }`
* `varName as predicate @filter(...) { ... }`

类型: `uid`

在查询中一个位置匹配的节点的`uid`可以存储在一个变量中，并在其它地方使用。查询变量可以在其他查询块中使用，也可以在定义块的子节点中使用。

查询变量在定义点不会影响查询的语义。查询变量被计算为所有与定义块匹配的节点。

通常，查询块是并行执行的，但是变量对一些块施加了计算顺序。由变量相关引起的循环是不允许的。

如果定义了变量，则必须在查询的其他地方使用它。

查询变量通过`uid(var-name)`提取其中的`uid`来使用。

语法`func: uid(A, B)`或`@filter(uid(A, B))`表示变量`A`和`B`的`uid`的并集。

查询示例：`angelina jolie`和`brad pitt`都出演过同一类型的电影。请注意，`B`和`D`匹配所有电影的所有类型，而不是每部电影的类型：

``` dql
{
 var(func:allofterms(name@en, "angelina jolie")) {
   actor.film {
    A AS performance.film {  # All films acted in by Angelina Jolie
     B As genre  # Genres of all the films acted in by Angelina Jolie
    }
   }
  }

 var(func:allofterms(name@en, "brad pitt")) {
   actor.film {
    C AS performance.film {  # All films acted in by Brad Pitt
     D as genre  # Genres of all the films acted in by Brad Pitt
    }
   }
  }

 films(func: uid(D)) @filter(uid(B)) {   # Genres from both Angelina and Brad
  name@en
   ~genre @filter(uid(A, C)) {  # Movies in either A or C.
     name@en
   }
 }
}
```

``` json 
{
  "data": {
    "films": [
      {
        "name@en": "War film",
        "~genre": [
          {
            "name@en": "Two-Fisted Tales"
          },
          {
            "name@en": "Alexander"
          }
          ...
        ]
      },
      {
        "name@en": "Indie film",
        "~genre": [
          {
            "name@en": "Babel"
          },
          {
            "name@en": "Seven"
          },
          {
            "name@en": "The Tree of Life"
          }
          ...
        ]
      }
      ...
    ]
  }
}
```

