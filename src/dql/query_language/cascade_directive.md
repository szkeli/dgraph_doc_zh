# 级联指令

使用`@cascade`指令，没有在查询中指定所有谓词的节点将被删除。在应用了某些筛选器或节点可能没有列出所有谓词的情况下，这可能很有用。

查询例：《哈利波特》电影，每个演员和角色都扮演过。使用`cascade`，任何不是由名为沃里克的演员扮演的角色都会被删除，就像任何没有名为沃里克的演员的《哈利波特》电影一样。如果没有“级联”，所有角色都会回归，但只有那些由名为沃里克的演员扮演的角色才会有演员的名字：

``` graphql
{
  HP(func: allofterms(name@en, "Harry Potter")) @cascade {
    name@en
    starring{
        performance.character {
          name@en
        }
        performance.actor @filter(allofterms(name@en, "Warwick")){
            name@en
         }
    }
  }
}
```

``` json 
{
  "data": {
    "HP": [
      {
        "name@en": "Harry Potter and the Chamber of Secrets",
        "starring": [
          {
            "performance.character": [
              {
                "name@en": "Professor Filius Flitwick"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          }
        ]
      },
      {
        "name@en": "Harry Potter and the Deathly Hallows - Part I",
        "starring": [
          {
            "performance.character": [
              {
                "name@en": "Griphook"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          }
        ]
      },
      {
        "name@en": "Harry Potter and the Deathly Hallows – Part 2",
        "starring": [
          {
            "performance.character": [
              {
                "name@en": "Professor Filius Flitwick"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          },
          {
            "performance.character": [
              {
                "name@en": "Griphook"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          }
        ]
      },
      {
        "name@en": "Harry Potter and the Prisoner of Azkaban",
        "starring": [
          {
            "performance.character": [
              {
                "name@en": "Professor Filius Flitwick"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          }
        ]
      },
      {
        "name@en": "Harry Potter and the Order of the Phoenix",
        "starring": [
          {
            "performance.character": [
              {
                "name@en": "Professor Filius Flitwick"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          }
        ]
      },
      {
        "name@en": "Harry Potter and the Goblet of Fire",
        "starring": [
          {
            "performance.character": [
              {
                "name@en": "Professor Filius Flitwick"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          }
        ]
      },
      {
        "name@en": "Harry Potter and the Half-Blood Prince",
        "starring": [
          {
            "performance.character": [
              {
                "name@en": "Professor Filius Flitwick"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          }
        ]
      },
      {
        "name@en": "Harry Potter and the Philosopher's Stone",
        "starring": [
          {
            "performance.character": [
              {
                "name@en": "Goblin Bank Teller"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          },
          {
            "performance.character": [
              {
                "name@en": "Professor Filius Flitwick"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          }
        ]
      }
    ]
  }
}
```

您也可以在内部查询块上应用`@cascade`：

``` graphql
{
  HP(func: allofterms(name@en, "Harry Potter")) {
    name@en
    genre {
      name@en
    }
    starring @cascade {
        performance.character {
          name@en
        }
        performance.actor @filter(allofterms(name@en, "Warwick")){
            name@en
         }
    }
  }
}
```

``` json 
{
  "data": {
    "HP": [
      {
        "name@en": "Harry Potter and the Chamber of Secrets",
        "genre": [
          {
            "name@en": "Family"
          },
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Fantasy"
          },
          {
            "name@en": "Mystery"
          }
        ],
        "starring": [
          {
            "performance.character": [
              {
                "name@en": "Professor Filius Flitwick"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          }
        ]
      },
      {
        "name@en": "Harry Potter and the Deathly Hallows - Part I",
        "genre": [
          {
            "name@en": "Fiction"
          },
          {
            "name@en": "Family"
          },
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Drama"
          },
          {
            "name@en": "Fantasy"
          },
          {
            "name@en": "Action Film"
          },
          {
            "name@en": "Mystery"
          }
        ],
        "starring": [
          {
            "performance.character": [
              {
                "name@en": "Griphook"
              }
            ],
            "performance.actor": [
              {
                "name@en": "Warwick Davis"
              }
            ]
          }
        ]
      }
      ...
    ]
  }
}
```

## 参数化@cascade

`@cascade`指令可以选择一个字段列表作为参数。这将更改默认行为，只考虑提供的字段，而不是类型的所有字段。列出的字段自动级联为嵌套选择集的必需参数。参数化的级联作用于层次(例如根函数或更低的层次)，所以你需要在你想要应用它的确切层次上指定`@cascade(param)`。

> 提示：`@cascade(predicate)`的规则是，谓词需要在查询中处于与`@cascade`相同的级别。

以下查询为例：

``` graphql
{
  nodes(func: allofterms(name@en, "jones indiana")) {
    name@en
    genre @filter(anyofterms(name@en, "action adventure")) {
      name@en
    }
    produced_by {
      name@en
    }
  }
}
```

``` json 
{
  "data": {
    "nodes": [
      {
        "name@en": "The Adventures of Young Indiana Jones: Passion for Life"
      },
      {
        "name@en": "Indiana Jones and the Temple of Doom",
        "genre": [
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Action/Adventure"
          },
          {
            "name@en": "Action Film"
          },
          {
            "name@en": "Costume Adventure"
          }
        ],
        "produced_by": [
          {
            "name@en": "Robert Watts"
          }
        ]
      },
      {
        "name@en": "The Adventures of Young Indiana Jones: The Perils of Cupid"
      },
      {
        "name@en": "The Adventures of Young Indiana Jones: Daredevils of the Desert",
        "genre": [
          {
            "name@en": "Adventure Film"
          }
        ]
      }
      ...
    ]
  }
}
```

该查询获取包含所有术语`jones indiana`的节点，然后遍历`genre`和`produced_by`。它还为游戏类型添加了一个额外的筛选条件，即只能获得名称中包含“动作”或“冒险”内容的游戏。结果包括没有类型的节点和没有类型和制作人的节点。

如果你使用没有参数的常规@级联，你就会失去那些有题材但没有制作人的游戏。

要获得具有遍历类型但可能没有`produced_by`的节点，你可以参数化级联：

``` graphql
{
  nodes(func: allofterms(name@en, "jones indiana")) @cascade(genre) {
    name@en
    genre @filter(anyofterms(name@en, "action adventure")) {
      name@en
    }
    produced_by {
      name@en
    }
    written_by {
      name@en
    }
  }
}
```

``` json 
{
  "data": {
    "nodes": [
      {
        "name@en": "Indiana Jones and the Temple of Doom",
        "genre": [
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Action/Adventure"
          },
          {
            "name@en": "Action Film"
          },
          {
            "name@en": "Costume Adventure"
          }
        ],
        "produced_by": [
          {
            "name@en": "Robert Watts"
          }
        ],
        "written_by": [
          {
            "name@en": "Gloria Katz"
          },
          {
            "name@en": "Willard Huyck"
          }
        ]
      }
      ...
    ]
  }
}
```

如果您想检查多个字段，只需用逗号分隔它们。例如，级联produced_by和written_by:

``` graphql
{
  nodes(func: allofterms(name@en, "jones indiana")) @cascade(produced_by,written_by) {
    name@en
    genre @filter(anyofterms(name@en, "action adventure")) {
      name@en
    }
    produced_by {
      name@en
    }
    written_by {
      name@en
    }
  }
}
```

``` json 
{
  "data": {
    "nodes": [
      {
        "name@en": "Indiana Jones and the Temple of Doom",
        "genre": [
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Action/Adventure"
          },
          {
            "name@en": "Action Film"
          },
          {
            "name@en": "Costume Adventure"
          }
        ],
        "produced_by": [
          {
            "name@en": "Robert Watts"
          }
        ],
        "written_by": [
          {
            "name@en": "Gloria Katz"
          },
          {
            "name@en": "Willard Huyck"
          }
        ]
      },
      {
        "name@en": "Indiana Jones and the Raiders of the Lost Ark",
        "genre": [
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Action Film"
          }
        ],
        "produced_by": [
          {
            "name@en": "Frank Marshall"
          }
        ],
        "written_by": [
          {
            "name@en": "Lawrence Kasdan"
          }
        ]
      },
      {
        "name@en": "Indiana Jones and the Kingdom of the Crystal Skull",
        "genre": [
          {
            "name@en": "Adventure Comedy"
          },
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Action Film"
          },
          {
            "name@en": "Costume Adventure"
          }
        ],
        "produced_by": [
          {
            "name@en": "Frank Marshall"
          },
          {
            "name@en": "Flávio R. Tambellini"
          }
        ],
        "written_by": [
          {
            "name@en": "David Koepp"
          }
        ]
      }
      ...
    ]
  }
}
```

### 嵌套和参数化级联

字段选择的级联特性被嵌套的@cascade覆盖。

前面的示例也可以沿链级联，并根据需要在每个级别上重写。

例如，如果你只想要“由制作《侏罗纪世界》的同一个人制作的印第安纳·琼斯电影”:

``` graphql
{
  nodes(func: allofterms(name@en, "jones indiana")) @cascade(produced_by) {
    name@en
    genre @filter(anyofterms(name@en, "action adventure")) {
      name@en
    }
    produced_by @cascade(producer.film) {
      name@en
      producer.film @filter(allofterms(name@en, "jurassic world")) {
        name@en
      }
    }
    written_by {
      name@en
    }
  }
}
```

``` json 
{
  "data": {
    "nodes": [
      {
        "name@en": "Indiana Jones and the Raiders of the Lost Ark",
        "genre": [
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Action Film"
          }
        ],
        "produced_by": [
          {
            "name@en": "Frank Marshall",
            "producer.film": [
              {
                "name@en": "Jurassic World"
              }
            ]
          }
        ],
        "written_by": [
          {
            "name@en": "Lawrence Kasdan"
          }
        ]
      },
      {
        "name@en": "Indiana Jones and the Kingdom of the Crystal Skull",
        "genre": [
          {
            "name@en": "Adventure Comedy"
          },
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Action Film"
          },
          {
            "name@en": "Costume Adventure"
          }
        ],
        "produced_by": [
          {
            "name@en": "Frank Marshall",
            "producer.film": [
              {
                "name@en": "Jurassic World"
              }
            ]
          }
        ],
        "written_by": [
          {
            "name@en": "David Koepp"
          }
        ]
      }
    ]
  }
}
```

另一个嵌套的例子：找到《星球大战》和《侏罗纪世界》的编剧和制片人是同一个人的《印第安纳琼斯》电影：

``` graphql
{
  nodes(func: allofterms(name@en, "jones indiana")) @cascade(produced_by,written_by) {
    name@en
    genre @filter(anyofterms(name@en, "action adventure")) {
      name@en
    }
    produced_by @cascade(producer.film) {
      name@en
      producer.film @filter(allofterms(name@en, "jurassic world")) {
        name@en
      }
    }
    written_by @cascade(writer.film) {
      name@en
      writer.film @filter(allofterms(name@en, "star wars")) {
        name@en
      }
    }
  }
}
```

``` json 
{
  "data": {
    "nodes": [
      {
        "name@en": "Indiana Jones and the Raiders of the Lost Ark",
        "genre": [
          {
            "name@en": "Adventure Film"
          },
          {
            "name@en": "Action Film"
          }
        ],
        "produced_by": [
          {
            "name@en": "Frank Marshall",
            "producer.film": [
              {
                "name@en": "Jurassic World"
              }
            ]
          }
        ],
        "written_by": [
          {
            "name@en": "Lawrence Kasdan",
            "writer.film": [
              {
                "name@en": "Star Wars Episode V: The Empire Strikes Back"
              },
              {
                "name@en": "Star Wars: The Force Awakens"
              }
            ]
          }
        ]
      }
    ]
  }
}
```
