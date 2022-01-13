# Pagination 分页

分页允许你只获取一部分而不是整个结果集。这对于`top-k`样式的查询以及减少客户端处理的结果集的大小或允许对结果进行分页访问都很有用。

分页通常与排序一起使用。

> 如果没有指定排序顺序，结果将按照uid进行排序，uid是随机分配的。因此，虽然顺序是确定的，但可能不是你所期望的。

## First 参数

语法示例：

* `q(func: ..., first: N)`
* `predicate (first: N) { ... }`
* `predicate @filter(...) (first: N) { ... }`

对于`N > 0`, `first: N`按排序或`UID`顺序检索前`N`个结果。

对于`N < 0`, `first: N`按排序或`UID`顺序检索最后`N`个结果。目前，只有当没有应用顺序时，才支持负数。为了在排序中获得负数的效果，颠倒排序的顺序并使用正的`N`。

查询示例：最后两部电影，按UID排序，由史蒂文·斯皮尔伯格执导，前三种类型的电影，按英文名的字母顺序排序：

``` graphql
{
  me(func: allofterms(name@en, "Steven Spielberg")) {
    director.film (first: -2) {
      name@en
      initial_release_date
      genre (orderasc: name@en) (first: 3) {
          name@en
      }
    }
  }
}
```

将返回：

``` json 
{
  "data": {
    "me": [
      {
        "director.film": [
          {
            "name@en": "Firelight",
            "initial_release_date": "1964-03-24T00:00:00Z",
            "genre": [
              {
                "name@en": "Science Fiction"
              },
              {
                "name@en": "Thriller"
              }
            ]
          },
          {
            "name@en": "Amazing Stories: Book One",
            "genre": [
              {
                "name@en": "Comedy"
              },
              {
                "name@en": "Drama"
              }
            ]
          }
        ]
      }
    ]
  }
}
```

查询例：在所有叫史蒂文的导演中，导演演员最多的三位叫史蒂文的导演：

``` dql
{
  ID as var(func: allofterms(name@en, "Steven")) @filter(has(director.film)) {
    director.film {
      stars as count(starring)
    }
    totalActors as sum(val(stars))
  }

  mostStars(func: uid(ID), orderdesc: val(totalActors), first: 3) {
    name@en
    stars : val(totalActors)

    director.film {
      name@en
    }
  }
}
```

将返回：

``` json
{
  "data": {
    "mostStars": [
      {
        "name@en": "Steven Spielberg",
        "stars": 1665,
        "director.film": [
          {
            "name@en": "Hook"
          },
          {
            "name@en": "The Color Purple"
          },
          {
            "name@en": "Schindler's List"
          },
          {
            "name@en": "Amistad"
          }
          ...
        ]
      }
    ]
  }
}
```

## Offset 参数

语法示例：

* `q(func: ..., offset: N)`
* `predicate (offset: N) { ... }`
* `predicate (first: M, offset: N) { ... }`
* `predicate @filter(...) (offset: N) { ... }`

通过指定`offset: N`，查询结果的前`N`就不会返回在结果集中。结合前面介绍的`first: M`使用，例如`first: M, offset: N`，就能跳过前面`N`个结果，只返回接下来的`M`个结果。

查询示例：查询英文名字为`Hark Tsui`的影片，跳过前面`4`个，返回接下来的`6`个。

``` dql
{
  me(func: allofterms(name@en, "Hark Tsui")) {
    name@zh
    name@en
    director.film (orderasc: name@en) (first:6, offset:4)  {
      genre {
        name@en
      }
      name@zh
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
        "name@zh": "徐克",
        "name@en": "Tsui Hark",
        "director.film": [
          {
            "genre": [
              {
                "name@en": "Science Fiction"
              },
              {
                "name@en": "Superhero movie"
              },
              {
                "name@en": "World cinema"
              },
              {
                "name@en": "Action/Adventure"
              },
              {
                "name@en": "Chinese Movies"
              },
              {
                "name@en": "Martial Arts Film"
              },
              {
                "name@en": "Action Film"
              }
            ],
            "name@en": "Black Mask 2: City of Masks",
            "initial_release_date": "2002-01-01T00:00:00Z"
          },
          {
            "genre": [
              {
                "name@en": "Adventure game"
              },
              {
                "name@en": "Drama"
              },
              {
                "name@en": "Action Film"
              }
            ],
            "name@zh": "狄仁杰之通天帝国",
            "name@en": "Detective Dee: Mystery of the Phantom Flame",
            "initial_release_date": "2010-09-05T00:00:00Z"
          },
          {
            "genre": [
              {
                "name@en": "Thriller"
              },
              {
                "name@en": "Crime Fiction"
              },
              {
                "name@en": "Action Film"
              }
            ],
            "name@en": "Don't Play with Fire",
            "initial_release_date": "1980-12-04T00:00:00Z"
          },
          {
            "genre": [
              {
                "name@en": "Thriller"
              },
              {
                "name@en": "Action/Adventure"
              },
              {
                "name@en": "Martial Arts Film"
              },
              {
                "name@en": "Action Thriller"
              },
              {
                "name@en": "Action Film"
              },
              {
                "name@en": "Buddy film"
              }
            ],
            "name@en": "Double Team",
            "initial_release_date": "1997-04-04T00:00:00Z"
          },
          {
            "genre": [
              {
                "name@en": "Adventure Film"
              },
              {
                "name@en": "Drama"
              },
              {
                "name@en": "Martial Arts Film"
              },
              {
                "name@en": "Action Film"
              }
            ],
            "name@zh": "龙门飞甲",
            "name@en": "Flying Swords of Dragon Gate",
            "initial_release_date": "2011-12-15T00:00:00Z"
          },
          {
            "genre": [
              {
                "name@en": "World cinema"
              },
              {
                "name@en": "Adventure Film"
              },
              {
                "name@en": "Romance Film"
              },
              {
                "name@en": "Action/Adventure"
              },
              {
                "name@en": "Drama"
              },
              {
                "name@en": "Fantasy"
              },
              {
                "name@en": "Chinese Movies"
              },
              {
                "name@en": "Romantic fantasy"
              },
              {
                "name@en": "Costume Adventure"
              },
              {
                "name@en": "Fantasy Adventure"
              }
            ],
            "name@zh": "青蛇",
            "name@en": "Green Snake",
            "initial_release_date": "1993-01-01T00:00:00Z"
          }
        ]
      },
      {
        "name@zh": "小倩",
        "name@en": "A Chinese Ghost Story: The Tsui Hark Animation"
      },
      {
        "name@en": "Tsui Hark Movie Studio"
      }
    ]
  }
}
```

## After 参数

语法示例：

* `q(func: ..., after: UID)`
* `predicate (first: N, after: UID) { ... }`
* `predicate @filter(...) (first: N, after: UID) { ... }`

另一个具有类似跳过功能的是使用默认`UID`排序时，能够直接指定`after: UID`，然后获取该`UID`之后的查询结果。例如，在一个常见的场景中，第一个查询是`predicate (after: 0x00, first: N)`或`predicate (first: N)`的形式，接下来的查询都用`predicate (after: <上一次查询的结果中的uid>, first: N)`。

查询示例：查询`Baz Luhrmann`的前`5`部影片，以`UID`排序：

``` dql
{
  me(func: allofterms(name@en, "Baz Luhrmann")) {
    name@en
    director.film (first:5) {
      uid
      name@en
    }
  }
}
```

``` json
{
  "data": {
    "me": [
      {
        "name@en": "Baz Luhrmann",
        "director.film": [
          {
            "uid": "0x434",
            "name@en": "The Great Gatsby"
          },
          {
            "uid": "0x1e0d",
            "name@en": "Strictly Ballroom"
          },
          {
            "uid": "0x11bdb",
            "name@en": "Moulin Rouge!"
          },
          {
            "uid": "0x27c60",
            "name@en": "Romeo + Juliet"
          },
          {
            "uid": "0x2e760",
            "name@en": "Australia"
          }
        ]
      }
    ]
  }
}
```

第五部电影是澳大利亚经典电影`Strictly Ballroom`。它的`UID`是`0x99e44`。在`Strictly Ballroom`影片之后的电影可以通过携带`after: N`参数的查询获取：

``` dql
{
  me(func: allofterms(name@en, "Baz Luhrmann")) {
    name@en
    director.film (first:5, after: 0x99e44) {
      uid
      name@en
    }
  }
}
```

``` json 
{
  "data": {
    "me": [
      {
        "name@en": "Baz Luhrmann",
        "director.film": [
          {
            "uid": "0xce247",
            "name@en": "Puccini: La Boheme (Sydney Opera)"
          },
          {
            "uid": "0xe40e5",
            "name@en": "No. 5 the Film"
          }
        ]
      }
    ]
  }
}
```