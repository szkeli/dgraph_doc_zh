# ignorereflex 指令

`@ignorereflex`指令强制通过查询结果中的任何路径删除作为父节点可访问的子节点

查询示例：`Rutger Hauer`的所有合作者。如果没有`@ignorereflex`，结果也会包括`Rutger Hauer`每一部电影：

``` graphql
{
  coactors(func: eq(name@en, "Rutger Hauer")) @ignorereflex {
    actor.film {
      performance.film {
        starring {
          performance.actor {
            name@en
          }
        }
      }
    }
  }
}
``` 

``` json 
{
  "data": {
    "coactors": [
      {
        "actor.film": [
          {
            "performance.film": [
              {
                "starring": [
                  {
                    "performance.actor": [
                      {
                        "name@en": "John de Lancie"
                      }
                    ]
                  },
                  {
                    "performance.actor": [
                      {
                        "name@en": "Lauren Lee Smith"
                      }
                    ]
                  },
                  {
                    "performance.actor": [
                      {
                        "name@en": "Dan Callahan"
                      }
                    ]
                  }
                  ...
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

