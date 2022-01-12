# 使用外部ID

`Dgraph`的输入语言`RDF`也支持`<a_fixed_identifier> <predicate> `literal/node`的三元组及其变体，其中标签`a_fixed_identifier`是一个节点的唯一标识符。例如，混合使用[schema.org](http://schema.org/)标识符、[电影数据库](https://www.themoviedb.org/)标识符和空节点：

``` rdf
_:userA <http://schema.org/type> <http://schema.org/Person> .
_:userA <dgraph.type> "Person" .
_:userA <http://schema.org/name> "FirstName LastName" .
<https://www.themoviedb.org/person/32-robin-wright> <http://schema.org/type> <http://schema.org/Person> .
<https://www.themoviedb.org/person/32-robin-wright> <http://schema.org/name> "Robin Wright" .
```

`Dgraph`原生不支持节点标识符这样的外部`id`，但外部`id`可以被存储为带有`xid`边的节点的属性。例如，在上面的例子中，谓词名称在`Dgraph`中是有效的，但是用`<http: schema.org="" person="">`标识的节点可以用`UID(比如0x123)`和一条边在`Dgraph`中标识：

``` rdf
<0x123> <xid> "http://schema.org/Person" .
<0x123> <dgraph.type> "ExternalType" .
```

而`Robin Wright`可能获得的`UID`是`0x321`和三元组：

``` rdf
<0x321> <xid> "https://www.themoviedb.org/person/32-robin-wright" .
<0x321> <http://schema.org/type> <0x123> .
<0x321> <http://schema.org/name> "Robin Wright" .
<0x321> <dgraph.type> "Person" .
```

相应的`Schema`应该是下面这样子的：

``` schema 
xid: string @index(exact) .
<http://schema.org/type>: [uid] @reverse .
```

查询所有人的例子：

``` dql
{
  var(func: eq(xid, "http://schema.org/Person")) {
    allPeople as <~http://schema.org/type>
  }

  q(func: uid(allPeople)) {
    <http://schema.org/name>
  }
}
```

通过外部`ID`查询`Robin Wright`的例子：

``` dql
{
  robin(func: eq(xid, "https://www.themoviedb.org/person/32-robin-wright")) {
    expand(_all_) { expand(_all_) }
  }
}

```

> 注意：`xid`边不会在变更中自动添加。一般来说，用户应该自己管理`xid`，并在必要时添加节点和`xid`边。`Dgraph`将所有此类`xid`的唯一性检查留给外部程序。