# 规范指令

使用`@normalize`指令，只返回有别名的谓词，并将结果平化以删除嵌套。

查询示例：`Steven Spielberg`每部电影的电影名，国家和前两位演员(按`UID`顺序)，没有`initial_release_date`，因为没有给出别名并通过`@normalize`进行平化：

``` dql
{
  director(func:allofterms(name@en, "steven spielberg")) @normalize {
    director: name@en
    director.film {
      film: name@en
      initial_release_date
      starring(first: 2) {
        performance.actor {
          actor: name@en
        }
        performance.character {
          character: name@en
        }
      }
      country {
        country: name@en
      }
    }
  }
}
```

``` json 
{
  "data": {
    "director": [
      {
        "director": "Steven Spielberg",
        "film": "Hook",
        "actor": "John Michael",
        "character": "Doctor",
        "country": "United States of America"
      },
      {
        "director": "Steven Spielberg",
        "film": "Hook",
        "actor": "Brad Parker",
        "character": "Jim",
        "country": "United States of America"
      },
      {
        "director": "Steven Spielberg",
        "film": "The Color Purple",
        "actor": "Carl Anderson",
        "country": "United States of America"
      },
      {
        "director": "Steven Spielberg",
        "film": "The Color Purple",
        "actor": "Oprah Winfrey",
        "character": "Sofia",
        "country": "United States of America"
      }
      ...
    ]
  }
}
```

还可以在嵌套查询块上应用`@normalize`。它将以类似的方式工作，但只是使应用了`@normalize`的嵌套查询块的结果变平。`@normalize`将返回一个列表，不管它应用在哪个类型的属性上：

``` dql
{
  director(func:allofterms(name@en, "steven spielberg")) {
    director: name@en
    director.film {
      film: name@en
      initial_release_date
      starring(first: 2) @normalize {
        performance.actor {
          actor: name@en
        }
        performance.character {
          character: name@en
        }
      }
      country {
        country: name@en
      }
    }
  }
}
```

``` json 
{
  "data": {
    "director": [
      {
        "director": "Steven Spielberg",
        "director.film": [
          {
            "film": "Hook",
            "initial_release_date": "1991-12-08T00:00:00Z",
            "starring": [
              {
                "actor": "John Michael",
                "character": "Doctor"
              },
              {
                "actor": "Brad Parker",
                "character": "Jim"
              }
            ],
            "country": [
              {
                "country": "United States of America"
              }
            ]
          },
          {
            "film": "The Color Purple",
            "initial_release_date": "1985-12-16T00:00:00Z",
            "starring": [
              {
                "actor": "Carl Anderson"
              },
              {
                "actor": "Oprah Winfrey",
                "character": "Sofia"
              }
            ],
            "country": [
              {
                "country": "United States of America"
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

