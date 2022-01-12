# RDF中的边属性 (Facet Lists in RDF)

Schema: 

``` schema
<name>: string @index(exact).
<nickname>: [string] .
```

用RDF创建包含facet的列表非常简单：

``` dql 
{
  set {
    _:Julian <name> "Julian" .
    _:Julian <nickname> "Jay-Jay" (kind="first") .
    _:Julian <nickname> "Jules" (kind="official") .
    _:Julian <nickname> "JB" (kind="CS-GO") .
  }
}
```

``` dql
{
  q(func: eq(name,"Julian")){
    name
    nickname @facets
  }
}
```

返回：

``` json 
{
  "data": {
    "q": [
      {
        "name": "Julian",
        "nickname|kind": {
          "0": "first",
          "1": "official",
          "2": "CS-GO"
        },
        "nickname": [
          "Jay-Jay",
          "Jules",
          "JB"
        ]
      }
    ]
  }
}
```