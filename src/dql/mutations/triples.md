# 三元组 (triples)

一个变更通过`set`关键字添加三元组`triples`：

``` dql
{
  set {
    # 在这里添加三元组
  }
}
```

这里的三元组是W3C标准[RDF N-Quad](https://www.w3.org/TR/n-quads/)格式的三元组。

每个三元组都有这样的格式：

``` dql
<subject> <predicate> <object> .
```

上面的格式表明由`subject`标识的每一个节点(node)都通过一条有向边(predicate)连接到一个对象`object`实体。三元组的`subject`总是`graph`图数据库中的节点`node`，`object`可以是一个值，也可以是一个节点(node)(字面量)。

比如，下面这些三元组：

``` dql
<0x01> <name> "Alice" .
<0x01> <dgraph.type> "Person" .
```

表示图上`ID`为`0x01`的节点(node)有一个值为`Alice`的属性`name`

然后下面这个三元组：

``` dql
<0x01> <friend> <0x02> .
```

表示`ID`为`0x01`的节点`node`通过`friend`这条边(`edge`)连接到`ID`为`0x02`的节点(`node`)

`Dgraph`为变更中的每个空白节点创建一个惟一的64位标识符————节点的`UID`。变更可以包含一个空白节点作为`subject`或对象实体(`object`)的标识符(即是`UID`)，或者一个来自先前变更的已知`UID`。