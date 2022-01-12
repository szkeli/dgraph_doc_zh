# 多查询块

在单个查询中，可以设置多个不同名字的查询块，这些查询块之间并行执行，它们之间不需要任何关联。

查询示例：自`2008`年以来`Angelina Jolie`所有类型的电影，以及`Peter Jackson`的电影：

``` dql
{
 AngelinaInfo(func:allofterms(name@en, "angelina jolie")) {
  name@en
   actor.film {
    performance.film {
      genre {
        name@en
      }
    }
   }
  }

 DirectorInfo(func: eq(name@en, "Peter Jackson")) {
    name@en
    director.film @filter(ge(initial_release_date, "2008"))  {
        Release_date: initial_release_date
        Name: name@en
    }
  }
}
```

``` json 
{
  "data": {
    "AngelinaInfo": [
      {
        "name@en": "Angelina Jolie",
        "actor.film": [
          {
            "performance.film": [
              {
                "genre": [
                  {
                    "name@en": "Short Film"
                  }
                ]
              }
            ]
          },
          {
            "performance.film": [
              {
                "genre": [
                  {
                    "name@en": "Slice of life"
                  },
                  {
                    "name@en": "Romance Film"
                  },
                  {
                    "name@en": "LGBT"
                  },
                  {
                    "name@en": "Drama"
                  },
                  {
                    "name@en": "Comedy"
                  },
                  {
                    "name@en": "Comedy-drama"
                  },
                  {
                    "name@en": "Ensemble Film"
                  }
                ]
              }
            ]
          },
          {
            "performance.film": [
              {
                "genre": [
                  {
                    "name@en": "Animation"
                  },
                  {
                    "name@en": "Romance Film"
                  },
                  {
                    "name@en": "Drama"
                  },
                  {
                    "name@en": "Comedy"
                  }
                ]
              }
            ]
          }
          ...
        ]
      },
      {
        "name@en": "Angelina Jolie"
      },
      {
        "name@en": "Angelina Jolie Look-a-Like"
      }
    ],
    "DirectorInfo": [
      {
        "name@en": "Peter Jackson",
        "director.film": [
          {
            "Release_date": "2012-11-28T00:00:00Z",
            "Name": "The Hobbit: An Unexpected Journey"
          },
          {
            "Release_date": "2009-11-24T00:00:00Z",
            "Name": "The Lovely Bones"
          }
          ... 
        ]
      },
      {
        "name@en": "Peter Jackson"
      },
      {
        "name@en": "Peter Jackson"
      },
      {
        "name@en": "Peter Jackson"
      }
    ]
  }
}
```

即使查询的答案中包含一些重叠部分，查询的结果集仍然是独立的。

查询示例：`Mackenzie Crook`演过的电影和`Jack Davenport`演过的电影，
两个结果集重叠，因为它们都在`Pirates of the Caribbean`电影中出演过，但结果是独立的，并且都包含完整的结果集：

``` dql
{
  Mackenzie(func:allofterms(name@en, "Mackenzie Crook")) {
    name@en
    actor.film {
      performance.film {
        uid
        name@en
      }
      performance.character {
        name@en
      }
    }
  }

  Jack(func:allofterms(name@en, "Jack Davenport")) {
    name@en
    actor.film {
      performance.film {
        uid
        name@en
      }
      performance.character {
        name@en
      }
    }
  }
}
```

``` json 
{
  "data": {
    "Mackenzie": [
      {
        "name@en": "Mackenzie Crook",
        "actor.film": [
          {
            "performance.film": [
              {
                "uid": "0x7f33f",
                "name@en": "Sex Lives of the Potato Men"
              }
            ]
          },
          {
            "performance.film": [
              {
                "uid": "0x5643",
                "name@en": "Still Crazy"
              }
            ],
            "performance.character": [
              {
                "name@en": "Dutch Kid"
              }
            ]
          },
          {
            "performance.film": [
              {
                "uid": "0x842bf",
                "name@en": "Solomon Kane"
              }
            ],
            "performance.character": [
              {
                "name@en": "Father Michael"
              }
            ]
          }
          ...
        ]
      }
    ],
    "Jack": [
      {
        "name@en": "Jack Davenport",
        "actor.film": [
          {
            "performance.film": [
              {
                "uid": "0x9367",
                "name@en": "The Libertine"
              }
            ],
            "performance.character": [
              {
                "name@en": "Harris"
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

## 变量块 `var block`

变量块由`var`关键字定义，不会返回在结果集，也不会影响查询的结果内容。

查询示例：`Angelina Jolie`的电影，通过流派排序：

``` graphql
{
  var(func: allofterms(name@en, "angelina jolie")) {
    name@en
    actor.film {
      A AS performance.film {
        B AS genre
      }
    }
  }

  films(func: uid(B), orderasc: name@en) {
    name@en
    ~genre @filter(uid(A)) {
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
        "name@en": "Action Film",
        "~genre": [
          {
            "name@en": "Gone in 60 Seconds"
          },
          {
            "name@en": "Alexander"
          }
          ...
        ]
      },
      {
        "name@en": "Action/Adventure",
        "~genre": [
          {
            "name@en": "Gone in 60 Seconds"
          },
          {
            "name@en": "The Bone Collector"
          }
          ...
        ]
      }
      ...
    ]
  }
}
```

## 多变量块 `multiple var block`

您还可以在单个查询操作中使用多个`var`块。你可以在任何后续的块中使用一个`var`块中的变量，但不能在同一个块中使用。

查询示例：包含`angelina jolie`和`morgan freeman`的电影(按名字排序)：

``` dql
{
  var(func:allofterms(name@en, "angelina jolie")) {
    name@en
    actor.film {
      A AS performance.film
    }
  }
  var(func:allofterms(name@en, "morgan freeman")) {
    name@en
    actor.film {
      B as performance.film @filter(uid(A))
    }
  }
  
  films(func: uid(B), orderasc: name@en) {
    name@en
  }
}
```

``` json 
{
  "data": {
    "films": [
      {
        "name@en": "Wanted"
      }
    ]
  }
}
```

## 在查询中组合使用多个变量块

你可以像下面这样在单个查询中使用逻辑连词组合多个变量块：

``` dql
{
  var(func:allofterms(name@en, "angelina jolie")) {
    name@en
    actor.film {
      A AS performance.film
    }
  }
  var(func:allofterms(name@en, "morgan freeman")) {
    name@en
    actor.film {
      B as performance.film
    }
  }
  films(func: uid(A,B), orderasc: name@en) @filter(uid(A) AND uid(B)) {
    name@en
  }
}
```

根查询`uid`函数将`var` `A`和`B`中的`uid`合并，因此您需要一个过滤器来与`var` `A`和`B`中的`uid`相交。



