# Fragement 片段

`fragment`关键字允许您根据`GraphQL`规范的`fragments`部分定义可以在查询中引用的新片段。片段允许重用常见的重复字段选择，从而减少DQL文档中的重复文本。片段可以嵌套在片段中，但在这种情况下不允许循环。例如：

``` bash
curl -H "Content-Type: application/dql" localhost:8080/query -XPOST -d $'
query {
  debug(func: uid(1)) {
    name@en
    ...TestFrag
  }
}
fragment TestFrag {
  initial_release_date
  ...TestFragB
}
fragment TestFragB {
  country
}' | python -m json.tool | less
```

> 注意GraphQL+-已重命名为Dgraph查询语言(DQL)。虽然application/dql是Content-Type头的首选值，但我们将继续支持Content-Type: application/graphql+-，以避免进行破坏性的更改。