# 分组 GroupBy

解析示例：
* `q(func: ...) @groupBy(predicate) { min(...) }`
* `predicate @groupBy(pred) { count(uid) }`

`groupby`查询聚合给定一组属性的查询结果，在这些属性上对元素进行分组。例如，一个包含块好友`@groupby(age) {count(uid)}`的查询，查找沿好友边缘可到达的所有节点，根据年龄将这些节点划分为组，然后计算每个组中有多少个节点。返回的结果是分组的边和聚合。

在一个`groupby`块中，只允许聚合，并且`count`只能应用于`uid`。

如果`groupby`应用于`uid`谓词，则生成的聚合可以保存在一个变量中(将分组的`uid`映射到聚合值)，并在查询的其他地方使用它来提取分组或聚合的边缘以外的信息。

查询示例：对于`Steven Spielberg`的电影，计算每种类型的电影数量，并为每种类型返回类型名称和数量。名称不能在`groupby`中提取，因为它不是一个聚合，但是`uid(a)`可以用于从`uid`到值映射中提取`uid`，从而按类型`uid`组织`byGenre`查询：

``` graphql
{
  var(func:allofterms(name@en, "steven spielberg")) {
    director.film @groupby(genre) {
      a as count(uid)
      # a is a genre UID to count value variable
    }
  }

  byGenre(func: uid(a), orderdesc: val(a)) {
    name@en
    total_movies : val(a)
  }
}
```

``` json 
{
  "data": {
    "byGenre": [
      {
        "name@en": "Drama",
        "total_movies": 21
      },
      {
        "name@en": "Adventure Film",
        "total_movies": 14
      },
      {
        "name@en": "Thriller",
        "total_movies": 13
      },
      {
        "name@en": "Action Film",
        "total_movies": 12
      }
      ...
    ]
  }
}
```

查询示例：蒂姆·伯顿电影中的演员以及他们在蒂姆·伯顿的电影中扮演了多少角色：

``` graphql
{
  var(func:allofterms(name@en, "Tim Burton")) {
    director.film {
      starring @groupby(performance.actor) {
        a as count(uid)
        # a is an actor UID to count value variable
      }
    }
  }

  byActor(func: uid(a), orderdesc: val(a)) {
    name@en
    val(a)
  }
}
```

``` json 
{
  "data": {
    "byActor": [
      {
        "name@en": "Johnny Depp",
        "val(a)": 8
      },
      {
        "name@en": "Helena Bonham Carter",
        "val(a)": 7
      }
      ...
    ]
  }
}
```