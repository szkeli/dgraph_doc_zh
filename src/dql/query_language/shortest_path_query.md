# 最短路径查询

通过查询块名称的关键字最短，可以找到源（`from`）节点和目的（`to`）节点之间的最短路径。它要求必须考虑遍历的源节点`UID`、目标节点`UID`和谓词（predicate）（至少一个）。最短查询块在查询响应中返回`_path_`下的最短路径。该路径也可以存储在其他查询块中使用的变量中。

## K-最短路径查询

默认情况下，返回的是最短路径。使用`numpaths`: `k`和`k > 1`，返回`k`最短路径。从`k`最短路径查询的结果中删除循环路径。对于`depth: n`，返回深度为`n`的路径。

> 请注意：
> * 如果在最短的块中没有指定谓词，则不会获取路径，因为没有遍历边界。
> * 如果查询需要很长时间，可以设置一个gRPC截止日期，在一段时间后停止查询。

例如：

``` bash 
curl localhost:8080/alter -XPOST -d $'
    name: string @index(exact) .
' | python -m json.tool | less
```


``` dql
{
  set {
    _:a <friend> _:b (weight=0.1) .
    _:b <friend> _:c (weight=0.2) .
    _:c <friend> _:d (weight=0.3) .
    _:a <friend> _:d (weight=1) .
    _:a <name> "Alice" .
    _:a <dgraph.type> "Person" .
    _:b <name> "Bob" .
    _:b <dgraph.type> "Person" .
    _:c <name> "Tom" .
    _:c <dgraph.type> "Person" .
    _:d <name> "Mallory" .
    _:d <dgraph.type> "Person" .
  }
}
```

`Alice`和`Mallory`之间的最短路径（假设`uid`分别为`0x2`和`0x5`）可以通过下面的查询找到：

``` dql
{
 path as shortest(from: 0x2, to: 0x5) {
  friend
 }
 path(func: uid(path)) {
   name
 }
}
```

> 注意，不考虑facet，每条边的权值为1

上面的`DQL`将返回：

``` json 
{
  "data": {
    "path": [
      {
        "name": "Alice"
      },
      {
        "name": "Mallory"
      }
    ],
    "_path_": [
      {
        "uid": "0x2",
        "friend": [
          {
            "uid": "0x5"
          }
        ]
      }
    ]
  }
}
```

通过指定`numpaths`，可以返回更多路径。设置`numpaths: 2`返回最短的两条路径：

``` dql
{

 A as var(func: eq(name, "Alice"))
 M as var(func: eq(name, "Mallory"))

 path as shortest(from: uid(A), to: uid(M), numpaths: 2) {
  friend
 }
 path(func: uid(path)) {
   name
 }
}
```

> 注意，在上面的查询中，我们使用`var`块和`UID()`函数来查询人，而不是使用`UID`字面量。还可以将它与`GraphQL`变量结合使用。

## 边的权重

`Dgraph`中的最短路径实现依赖于`facet`来提供权重。使用边(edge)上的facet可以让你定义如下的边(edge)的权重：

> 注意：在最短的查询块中，每个谓词(predicate)只允许有一个facet。

``` dql
{
 path as shortest(from: 0x2, to: 0x5) {
  friend @facets(weight)
 }

 path(func: uid(path)) {
  name
 }
}
```

``` json 
{
  "data": {
    "path": [
      {
        "name": "Alice"
      },
      {
        "name": "Bob"
      },
      {
        "name": "Tom"
      },
      {
        "name": "Mallory"
      }
    ],
    "_path_": [
      {
        "uid": "0x2",
        "friend": [
          {
            "uid": "0x3",
            "friend|weight": 0.1,
            "friend": [
              {
                "uid": "0x4",
                "friend|weight": 0.2,
                "friend": [
                  {
                    "uid": "0x5",
                    "friend|weight": 0.3
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

### 遍历的例子

下面是一个图遍历的例子，它允许您找到使用`Car`或`Bus`的朋友之间的最短路径：

> 每个关系的`Car`和`Bus`移动被建模为`facet`，并在最短的查询中指定

``` dql
{
  set {
    _:a <friend> _:b (weightCar=10, weightBus=1 ) .
    _:b <friend> _:c (weightCar=20, weightBus=1) .
    _:c <friend> _:d (weightCar=11, weightBus=1.1) .
    _:a <friend> _:d (weightCar=70, weightBus=2) .
    _:a <name> "Alice" .
    _:a <dgraph.type> "Person" .
    _:b <name> "Bob" .
    _:b <dgraph.type> "Person" .
    _:c <name> "Tom" .
    _:c <dgraph.type> "Person" .
    _:d <name> "Mallory" .
    _:d <dgraph.type> "Person" .
  }
}
```

依赖`Car`和`Bus`查询最短路径：

``` dql
{

 A as var(func: eq(name, "Alice"))
 M as var(func: eq(name, "Mallory"))

 sPathBus as shortest(from: uid(A), to: uid(M)) {  
  friend
  @facets(weightBus)
 }

 sPathCar as shortest(from: uid(A), to: uid(M)) {  
  friend
  @facets(weightCar)
 }  
  
 pathBus(func: uid(sPathBus)) {
   name   
 }
  
 pathCar(func: uid(sPathCar)) {
   name   
 }
}
```

响应包含以下符合指定权重的路径：

``` json 
 "pathBus": [
      {
        "name": "Alice"
      },
      {
        "name": "Mallory"
      }
    ],
    "pathCar": [
      {
        "name": "Alice"
      },
      {
        "name": "Bob"
      },
      {
        "name": "Tom"
      },
      {
        "name": "Mallory"
      }
    ]
```

## 约束

可以对中间节点应用如下约束：

``` dql
{
  path as shortest(from: 0x2, to: 0x5) {
    friend @filter(not eq(name, "Bob")) @facets(weight)
    relative @facets(liking)
  }

  relationship(func: uid(path)) {
    name
  }
}
```

k最短路径算法（在`numpaths > 1`时使用）也接受参数`minweight`和`maxweight`，它们的值为一个浮点数。当传递它们时，只有权重范围`[minweight, maxweight]`内的路径将被视为有效路径。例如，这可以用于查询在2到4个节点之间的最短路径：

``` dql
{
 path as shortest(from: 0x2, to: 0x5, numpaths: 2, minweight: 2, maxweight: 4) {
  friend
 }
 path(func: uid(path)) {
   name
 }
}
```

## 笔记

对于最短路径查询，需要记住以下几点:

* 权值必须是非负的。`Dijkstra`算法用于计算最短路径。
* 在最短的查询块中，每个谓词只允许有一个`facet`。
* 每个查询只允许一个最短路径块。结果中只返回一个`_path_`。对于`numpaths > 1`的查询，`_path_`包含所有路径。
* 循环路径不包含在k最短路径查询结果中。
* 对于`k`条最短路径(当`numpaths > 1`时)，最短路径查询变量的结果将只返回一条路径，这将是`k`条路径中的最短路径。所有`k`条路径都以`_path_`形式返回。

