# 空白节点和`UID`

变更中的空白节点(node)，写作`_:identifier`，用于标识变更中的节点。`Dgraph`为每一个空白节点创建一个`UID`以标识这个空白节点，并且在最后`Dgraph`会返回这些创建的`UID`集作为返回结果，例如，对于以下变更：

``` dql
{
 set {
    _:class <student> _:x .
    _:class <student> _:y .
    _:class <name> "awesome class" .
    _:class <dgraph.type> "Class" .
    _:x <name> "Alice" .
    _:x <dgraph.type> "Person" .
    _:x <dgraph.type> "Student" .
    _:x <planet> "Mars" .
    _:x <friend> _:y .
    _:y <name> "Bob" .
    _:y <dgraph.type> "Person" .
    _:y <dgraph.type> "Student" .
 }
}
```

返回结果(实际的`uid`在每次执行该变更时都是不同的)：


``` json 
{
  "data": {
    "code": "Success",
    "message": "Done",
    "uids": {
      "class": "0x2712",
      "x": "0x2713",
      "y": "0x2714"
    }
  }
}
```

至此，数据库就像存储了三元组一样被更新了：

``` dql 
<0x6bc818dc89e78754> <student> <0xc3bcc578868b719d> .
<0x6bc818dc89e78754> <student> <0xb294fb8464357b0a> .
<0x6bc818dc89e78754> <name> "awesome class" .
<0x6bc818dc89e78754> <dgraph.type> "Class" .
<0xc3bcc578868b719d> <name> "Alice" .
<0xc3bcc578868b719d> <dgraph.type> "Person" .
<0xc3bcc578868b719d> <dgraph.type> "Student" .
<0xc3bcc578868b719d> <planet> "Mars" .
<0xc3bcc578868b719d> <friend> <0xb294fb8464357b0a> .
<0xb294fb8464357b0a> <name> "Bob" .
<0xb294fb8464357b0a> <dgraph.type> "Person" .
<0xb294fb8464357b0a> <dgraph.type> "Student" .
```

空节点标签`_:class`、`_:x`和`_:y`不再标识变更成功后的节点，因此可以安全地重用它们来标识以后发生变更时的新节点。

以后的变化可以更新现有`uid`的数据。例如，下面将向`uid`为`0x6bc818dc89e78754`这个班级`class`中添加一个新学生：

``` dql 
{
 set {
    <0x6bc818dc89e78754> <student> _:x .
    _:x <name> "Chris" .
    _:x <dgraph.type> "Person" .
    _:x <dgraph.type> "Student" .
 }
}
```

查询也可以直接使用UID：

``` dql
{
 class(func: uid(0x6bc818dc89e78754)) {
  name
  student {
   name
   planet
   friend {
    name
   }
  }
 }
}
```