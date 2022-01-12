# 类型系统

`Dgraph`支持一个类型系统，该系统可用于对节点进行分类，并根据节点的类型查询节点。在展开(expand)查询期间也使用类型系统。

## 类型定义

类型是使用类似`graphql`的语法定义的。例如：

``` dql
type Student {
  name
  dob
  home_address
  year
  friends
}
```

> 注意，不能定义以dgraph开头的类型名。，它保留为`Dgraph`的内部类型/predicate的名称空间。例如，定义`dgraph.student`类型无效。

类型与模式一起使用`Alter`端点声明。为了正确支持上述类型，还需要为该类型中的每个属性指定一个`predicate`，例如：

``` dql
name: string @index(term) .
dob: datetime .
home_address: string .
year: int .
friends: [uid] .
```

反向谓词也可以包含在类型定义中。例如，如果有一个具有反向边的谓词子谓词(谓词名称周围的括号需要正确理解特殊字符~)，那么上面的类型可以扩展为包含student的父谓词。

``` dql
children: [uid] @reverse .

type Student {
  name
  dob
  home_address
  year
  friends
  <~children>
}
```

边可以用于多种类型:例如，名字可以同时用于人和宠物。但是，有时需要为每种类型使用不同的谓词来表示类似的概念。例如，如果学生名和图书名需要不同的索引，那么谓词必须不同。

``` dql
type Student {
  student_name
}

type Textbook {
  textbook_name
}

student_name: string @index(exact) .
textbook_name: string @lang @index(fulltext) .
```

更改已经存在的类型的模式，将覆盖现有的定义。

## 设置一个节点的类型

标量节点不能具有类型，因为它们只有一个属性，且其类型是节点的类型。`UID`节点可以有一个类型。通过设置`dgraph`的值来设置类型。该节点的类型`predicate`。一个节点可以有多个类型。下面是一个如何设置节点类型的例子：

``` dql
{
  set {
    _:a <name> "Garfield" .
    _:a <dgraph.type> "Pet" .
    _:a <dgraph.type> "Animal" .
  }
}
```

`dgraph.type`是保留谓词，不能删除或修改。

## 在查询过程中使用类型系统

可以在查询的顶层函数中使用类型系统：

``` dql 
{
  q(func: type(Animal)) {
    uid
    name
  }
}
```

这个查询将只返回类型设置为`Animal`的节点。

类型还可以用于过滤查询内部的结果。例如：

``` dql
{
  q(func: has(parent)) {
    uid
    parent @filter(type(Person)) {
      uid
      name
    }
  }
}
```

该查询将返回具有父谓词且只具有`Person`类型的父谓词的节点。

您还可以查询`dgraph.type`获取实体类型。例如：

``` dql
{
   people(func: eq(dgraph.type, "Person")) {
      name
      dgraph.type
   }
}
```

或者

``` dql
{
   people(func: type(Person)) {
      name
      dgraph.type
   }
}
```

## 删除一个类型

可以使用`Alter`端点删除类型定义。所有需要做的就是发送一个带有字段`DropOp`(或`drop_op`，取决于客户端)的操作对象到`enum`值`TYPE`，并将字段`DropValue`(或`drop_value`)发送到要删除的类型。

下面是一个使用`Go`客户端删除`Person`类型的例子：

``` go
err := c.Alter(context.Background(), &api.Operation {
    DropOp: api.Operation_TYPE,
    DropValue: "Person"})
```


## 展开查询与类型系统

使用`expand`的查询(例如:`expand(_all_)`)要求要扩展的节点具有类型。