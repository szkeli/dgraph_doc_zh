# 删除 (delete)

由`delete`关键字标识的删除变更能从数据库中删除三元组。

例如，如果该数据库包含以下内容：

``` rdf
<0xf11168064b01135b> <name> "Lewis Carrol"
<0xf11168064b01135b> <died> "1998"
<0xf11168064b01135b> <dgraph.type> "Person" .
```

然后，下面的`delete`变更删除指定谓语(predicate)的错误数据，并将其从任何索引中删除：

``` dql
{
  delete {
    <0xf11168064b01135b> <died> "1998" .
  }
}
```

## 通配符删除 (Wildcard Delete)

在许多情况下，你可能需要删除一个谓词(predicate)的多个数据。对于给定的节点`N`，谓词`P`的所有数据(以及所有对应的索引)都用模式`S P *`删除：

``` dql
{
  delete {
    <0xf11168064b01135b> <author.of> * .
  }
}
```

模式`S * *`删除节点中所有已知的边，与被删除的边对应的任何反向边，以及对被删除数据的任何索引：

> 注意：对于符合`S * *`模式的变更，只删除与给定节点(使用`dgraph.type`)关联的类型中的谓词。任何不匹配节点类型之一的谓词将在`S * *`删除变更后保留。

``` dql
{
  delete {
    <0xf11168064b01135b> * * .
  }
}
```

如果删除模式`S * *`中的节点`S`只有几条边带有`dgraph.type`定义，则只删除那些带有`dgraph.type`定义的三元组。包含非`dgraph.type`定义的边的节点在`S * *`删除变更之后仍然存在。

> 注意，`* P O`和`* * O`的删除模式不受支持，因为存储和查找所有传入边的效率很低。

## 非列表边的删除 (Deletion of non-list predicates)

删除非列表谓词(即1对1关系)的值可以通过两种方式完成：

* 使用上一节中提到的通配符删除(星号表示法)。
* 将对象设置为特定的值。如果传递的值不是当前值，则变更将成功，但不会产生任何影响。如果传递的值是当前值，则变更将成功，并将删除非列表谓词。

对于被语言标记的值，支持以下特殊语法：

``` dql 
{
  delete {
    <0x12345> <name@es> * .
  }
}
```

在本例中，使用语言标记`es`标记的`name`字段的值被删除，其他标记的值保持不变。