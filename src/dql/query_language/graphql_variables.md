# GraphQL 变量

解析规则(使用默认值)：

* `query title($name: string = "Bauman") { ... }`
* `query title($age: int = "95") { ... }`
* `query title($uids: string = "0x1") { ... }`
* `query title($uids: string = "[0x1, 0x2, 0x3]") { ... }`

可以在查询中定义和使用变量，这有助于查询重用，并通过传递单独的变量映射，避免运行时在客户端中构建代价高昂的字符串。变量以$符号开头。对于带有GraphQL变量的HTTP请求，我们必须使用`Content-Type: application/json`头，并通过包含查询和变量的json对象传递数据：

``` bash 
curl -H "Content-Type: application/json" localhost:8080/query -XPOST -d $'{
  "query": "query test($a: string) { test(func: eq(name, $a)) { \n uid \n name \n } }",
  "variables": { "$a": "Alice" }
}' | python -m json.tool | less
```

``` dql
query test($a: int, $b: int, $name: string) {
  me(func: allofterms(name@en, $name)) {
    name@en
    director.film (first: $a, offset: $b) {
      name @en
      genre(first: $a) {
        name@en
      }
    }
  }
}
```

``` json 
{
  "data": {
    "me": [
      {
        "name@en": "Steven Spielberg",
        "director.film": [
          {
            "name@en": "Close Encounters of the Third Kind",
            "genre": [
              {
                "name@en": "Science Fiction"
              },
              {
                "name@en": "Adventure Film"
              },
              {
                "name@en": "Drama"
              }
            ]
          }
          ...
        ]
      },
      {
        "name@en": "Steven Spielberg And The Return To Film School"
      },
      {
        "name@en": "Directors: Steven Spielberg"
      },
      {
        "name@en": "Steven Spielberg"
      }
      ...
    ]
  }
}
```

* 变量可以有默认值。在下面的例子中，$a的默认值为2。因为变量映射中没有提供$a的值，所以$a采用默认值。
* 类型以!不能有默认值，但必须有一个值作为变量映射的一部分。
* 变量的值必须对给定的类型具有可解析性，如果不能，则抛出错误。
* 目前支持的变量类型有:int、float、bool和string。
* 任何正在使用的变量都必须在开头的命名查询子句中声明。

``` dql
query test($a: int = 2, $b: int!, $name: string) {
  me(func: allofterms(name@en, $name)) {
    director.film (first: $a, offset: $b) {
      genre(first: $a) {
        name@en
      }
    }
  }
}
```

``` json 
{
  "data": {
    "me": [
      {
        "director.film": [
          {
            "genre": [
              {
                "name@en": "Science Fiction"
              },
              {
                "name@en": "Adventure Film"
              }
            ]
          },
          {
            "genre": [
              {
                "name@en": "Horror"
              },
              {
                "name@en": "Science Fiction"
              }
            ]
          }
        ]
      }
    ]
  }
}
```


你也可以使用数组与GraphQL变量：

``` dql
query test($a: int = 2, $b: int!, $aName: string, $bName: string) {
  me(func: eq(name@en, [$aName, $bName])) {
    director.film (first: $a, offset: $b) {
      genre(first: $a) {
        name@en
      }
    }
  }
}
```

``` json 
{
  "data": {
    "me": [
      {
        "director.film": [
          {
            "genre": [
              {
                "name@en": "Indie film"
              },
              {
                "name@en": "Thriller"
              }
            ]
          },
          {
            "genre": [
              {
                "name@en": "Farce"
              },
              {
                "name@en": "Comedy"
              }
            ]
          }
        ]
      },
      {
        "director.film": [
          {
            "genre": [
              {
                "name@en": "Science Fiction"
              },
              {
                "name@en": "Adventure Film"
              }
            ]
          },
          {
            "genre": [
              {
                "name@en": "Horror"
              },
              {
                "name@en": "Science Fiction"
              }
            ]
          }
        ]
      }
    ]
  }
}
```


我们还支持facet变量替换：

``` dql
query test($name: string = "Alice", $IsClose: string = "true") {
  data(func: eq(name, $name)) {
    friend @facets(eq(close, $IsClose)) {
      name
    }
      colleague : friend @facets(eq(close, false)) {
      name
    }
  }
}
```

``` json 
{
  "data": {
    "data": [
      {
        "friend": [
          {
            "name": "Bob"
          },
          {
            "name": "Dave"
          }
        ],
        "colleague": [
          {
            "name": "Charlie"
          }
        ]
      }
    ]
  }
}
```


