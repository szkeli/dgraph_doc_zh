# 反向边 (reverse edges)

`Dgraph`中的任何出边(`outgoing edge`)都可以使用`Schema`中的`@reverse`指令反向，并使用波浪号作为边名的前缀来查询。例如`<~myEdge>`。

`Dgraph`是有向图模型，这意味着所有属性总是在一个方向上从一个实体指向另一个实体或值，即`S P -> O`。

反向边是自动生成的边，不属于你的数据集的一部分。这意味着您不能直接在反向边上提交变更。改变正向边将会自动更新反向边的值。

**正确地使用反向边**

在`RDF`中，三元组的排列已经定义了可以逆转的内容：

``` rdf
_:MyObject <myEdge> _:BlankNode  . #反向边正确的语法
_:BlankNode <dgraph.type> "Person" .
```

纠正和应用反向边的最简单方法是使用`JSON`。它只是将`Schema`上的指令放在所需的边上。在构建你的变更时，请记住`JSON`中没有反向边相关的语法。因此，您应该做的事情与`RDF`类似：更改`JSON`对象的排列。

因为`MyObject`实体在`Person`实体之上，所以在格式化变更时，`MyObject`必须在前面：

``` json
{
   "set": [
      {
         "uid": "_:MyObject",
         "dgraph.type": "Object",
         "myEdge": {
            "uid": "_:BlankNode",
            "dgraph.type": "Person"
         }
      }
   ]
}
```

另一种方法是分割成一些小块或小批，并使用空白节点作为引用。这有助于组织和重用引用：

``` json
{
   "set": [
      {
         "uid": "_:MyObject",
         "dgraph.type": "Object",
         "myEdge": [{"uid": "_:BlankNode"}]
      },
      {
         "uid": "_:BlankNode",
         "dgraph.type": "Person"
      }
   ]
}
```

## 更多的关于反向边的小例子

在`RDF`中，想要正确地使用反向边非常简单：

``` rdf
name: String .
husband: uid @reverse .
wife: uid @reverse .
parent: [uid] @reverse .
```

``` schema
{
  set {
    _:Megalosaurus <name> "Earl Sneed Sinclair" .
    _:Megalosaurus <dgraph.type> "Dinosaur" .
    _:Megalosaurus <wife> _:Allosaurus .
    _:Allosaurus <name> "Francis Johanna Phillips Sinclair" (short="Fran") .
    _:Allosaurus <dgraph.type> "Dinosaur" .
    _:Allosaurus <husband> _:Megalosaurus .
    _:Hypsilophodon <name> "Robert Mark Sinclair" (short="Robbie") .
    _:Hypsilophodon <dgraph.type> "Dinosaur" .
    _:Hypsilophodon <parent> _:Allosaurus (role="son") .
    _:Hypsilophodon <parent> _:Megalosaurus (role="son") .
    _:Protoceratops <name> "Charlene Fiona Sinclair" .
    _:Protoceratops <dgraph.type> "Dinosaur" .
    _:Protoceratops <parent> _:Allosaurus (role="daughter") .
    _:Protoceratops <parent> _:Megalosaurus (role="daughter") .
    _:MegalosaurusBaby <name> "Baby Sinclair" (short="Baby") .
    _:MegalosaurusBaby <dgraph.type> "Dinosaur" .
    _:MegalosaurusBaby <parent> _:Allosaurus (role="son") .
    _:MegalosaurusBaby <parent> _:Megalosaurus (role="son") .
  }
}
```

边的方向应该如下：

``` text
Exchanged hierarchy:
 Object -> Parent;
 Object <~ Parent; #Reverse
 Children to parents via "parent" edge.
 wife and husband bidirectional using reverse.
Normal hierarchy:
 Parent -> Object;
 Parent <~ Object; #Reverse
 This hierarchy is not part of the example, but is generally used in all graph models.
 To make this hierarchy we need to bring the hierarchical relationship starting from the parents and not from the children. Instead of using the edges "wife" and "husband" we switch to single edge called "married" to simplify the model.
    _:Megalosaurus <name> "Earl Sneed Sinclair" .
    _:Megalosaurus <dgraph.type> "Dinosaur" .
    _:Megalosaurus <married> _:Allosaurus .
    _:Megalosaurus <parent> _:Hypsilophodon (role="son") .
    _:Megalosaurus <parent> _:Protoceratops (role="daughter") .
    _:Megalosaurus <parent> _:MegalosaurusBaby (role="son") .
    _:Allosaurus <name> "Francis Johanna Phillips Sinclair" (short="Fran") .
    _:Allosaurus <dgraph.type> "Dinosaur" .
    _:Allosaurus <married> _:Megalosaurus .
    _:Allosaurus <parent> _:Hypsilophodon (role="son") .
    _:Allosaurus <parent> _:Protoceratops (role="daughter") .
    _:Allosaurus <parent> _:MegalosaurusBaby (role="son") .
```

## 查询

1. `wife_husband`是`wife`的反向边。
2. `husband`是一条正常的正向边。

``` dql
{
  q(func: has(wife)) {
    name
    WF as wife {
      name
    }
  }
  reverseIt(func: uid(WF)) {
    name
    wife_husband : ~wife {
      name
    }
    husband {
      name
    }
  }
}
```

1. `Children` 是 `parent` 的反向边：

``` dql
{
  q(func: has(name)) @filter(eq(name, "Earl Sneed Sinclair")){
    name
    Children : ~parent @facets {
      name
    }
  }
}
```

## 反向边和 facet

反向边上的`facet`与正向边上的相同。也就是说，如果您在一条边上设置或更新一个`facet`，那么它的反向边将具有相同的`facet`：

``` dql
{
  set {
    _:Megalosaurus <name> "Earl Sneed Sinclair" .
    _:Megalosaurus <dgraph.type> "Dinosaur" .
    _:Megalosaurus <wife> _:Allosaurus .
    _:Megalosaurus <parent> _:MegalosaurusBaby (role="parent -> child") .
    _:MegalosaurusBaby <name> "Baby Sinclair" (short="Baby -> parent") .
    _:MegalosaurusBaby <dgraph.type> "Dinosaur" .
    _:MegalosaurusBaby <parent> _:Megalosaurus (role="child -> parent") .
  }
}
```

使用前面例子中的类似查询：

``` dql
{
  Parent(func: has(name)) @filter(eq(name, "Earl Sneed Sinclair")){
    name
    C as Children : parent @facets {
      name
    }
  }
    Child(func: uid(C)) {
      name
      parent @facets {
        name
      }
    }
}
```

``` json
{
  "data": {
    "Parent": [
      {
        "name": "Earl Sneed Sinclair",
        "Children": [
          {
            "name": "Baby Sinclair",
            "Children|role": "parent -> child"
          }
        ]
      }
    ],
    "Child": [
      {
        "name": "Baby Sinclair",
        "parent": [
          {
            "name": "Earl Sneed Sinclair",
            "parent|role": "child -> parent"
          }
        ]
      }
    ]
  }
}
```