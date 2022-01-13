# Connecting Filters 联合过滤

在`@filter`中，多个函数可以与布尔连接符一起使用。

## AND(与)、OR(或)、NOT(非)

连接词`AND`、`OR`和`NOT`连接过滤器，可以构建到任意复杂的过滤器中，如`(NOT A OR B)`和`(C AND NOT (D OR E))`。注意，`NOT`绑定比`AND`绑定更紧密，`AND`绑定比`OR`绑定更紧密。

查询示例：所有史蒂芬·斯皮尔伯格`(Steven Spielberg)`同时包含“indiana”和`jones`或`jurassic`和`park`的电影：

``` dql
{
  me(func: eq(name@en, "Steven Spielberg")) @filter(has(director.film)) {
    name@en
    director.film @filter(allofterms(name@en, "jones indiana") OR allofterms(name@en, "jurassic park"))  {
      uid
      name@en
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
        "name@en": "Steven Spielberg",
        "director.film": [
          {
            "uid": "0x9a41",
            "name@en": "Indiana Jones and the Temple of Doom"
          },
          {
            "uid": "0x1cde3",
            "name@en": "Jurassic Park"
          },
          {
            "uid": "0x2868c",
            "name@en": "Indiana Jones and the Raiders of the Lost Ark"
          },
          {
            "uid": "0x42164",
            "name@en": "Indiana Jones and the Kingdom of the Crystal Skull"
          },
          {
            "uid": "0x483f3",
            "name@en": "The Lost World: Jurassic Park"
          },
          {
            "uid": "0x54387",
            "name@en": "Indiana Jones and the Last Crusade"
          }
        ]
      }
    ]
  }
}
```


