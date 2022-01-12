# Count 计数

语法示例：
* `count(predicate)`
* `count(uid)`

`count(predicate)`计算节点的边的数量。

`count(uid)`统计在封闭块中匹配的`uid`的数量。

查询示例：每个演员以`Orlando`的名义出演的电影数量：

``` dql
{
  me(func: allofterms(name@en, "Orlando")) @filter(has(actor.film)) {
    name@en
    count(actor.film)
  }
}
```

``` json 
{
  "data": {
    "me": [
      {
        "name@en": "Orlando Seale",
        "count(actor.film)": 13
      },
      {
        "name@en": "Antonio Orlando",
        "count(actor.film)": 5
      },
      {
        "name@en": "Orlando Viera",
        "count(actor.film)": 1
      },
      {
        "name@en": "Silvio Orlando",
        "count(actor.film)": 32
      }
      ...
    ]
  }
}
```

`Count` 能用在根查询和[别名](./aliases.md)`Aliase`中。

查询示例：导演超过5部电影的导演数。在查询根处使用时，需要[count索引](./schema.md#count-indices)：

``` dql
{
  directors(func: gt(count(director.film), 5)) {
    totalDirectors : count(uid)
  }
}
```

``` json 
{
  "data": {
    "directors": [
      {
        "totalDirectors": 7712
      }
    ]
  }
}
```

可以将`Count`赋值给[值变量](./value_variables.md)：

查询示例：`Ang Lee`的`Eat Drink Man Woman`的演员按电影的数量返回结果：


``` dql
{
  var(func: allofterms(name@en, "eat drink man woman")) {
    starring {
      actors as performance.actor {
        totalRoles as count(actor.film)
      }
    }
  }

  edmw(func: uid(actors), orderdesc: val(totalRoles)) {
    name@en
    name@zh
    totalRoles : val(totalRoles)
  }
}
```

``` json
{
  "data": {
    "edmw": [
      {
        "name@en": "Sylvia Chang",
        "name@zh": "张艾嘉",
        "totalRoles": 35
      },
      {
        "name@en": "Chien-lien Wu",
        "name@zh": "吴倩莲",
        "totalRoles": 20
      },
      {
        "name@en": "Yang Kuei-mei",
        "name@zh": "杨贵媚",
        "totalRoles": 14
      },
      {
        "name@en": "Winston Chao",
        "name@zh": "赵文瑄",
        "totalRoles": 11
      }
      ...
    ]
  }
}
```
