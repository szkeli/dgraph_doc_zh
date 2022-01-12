# 递归查询

递归查询允许您遍历一组谓词(predicate)(使用过滤器、facet等)，直到我们到达所有叶子节点，或者到达由`depth`参数指定的最大深度。

要从一个拥有超过30000部电影的类型中获得10部电影，然后为这些电影找两个演员，我们会做如下事情：

``` dql
{
	me(func: gt(count(~genre), 30000), first: 1) @recurse(depth: 5, loop: true) {
		name@en
		~genre (first:10) @filter(gt(count(starring), 2))
		starring (first: 2)
		performance.actor
	}
}
```

``` json 
{
  "data": {
    "me": [
      {
        "name@en": "Documentary film",
        "~genre": [
          {
            "name@en": "Casting Crowns: The Altar and the Door Live",
            "starring": [
              {
                "performance.actor": [
                  {
                    "name@en": "Megan Garrett"
                  }
                ]
              },
              {
                "performance.actor": [
                  {
                    "name@en": "Chris Huffman"
                  }
                ]
              }
            ]
          },
          {
            "name@en": "Heavy Water: A Film for Chernobyl",
            "starring": [
              {
                "performance.actor": [
                  {
                    "name@en": "David Bickerstaff"
                  }
                ]
              },
              {
                "performance.actor": [
                  {
                    "name@en": "Phil Grabsky"
                  }
                ]
              }
            ]
          }
          ...
        ]
      }
    ]
  }
}
```

**使用递归查询时需要记住的几点是:**

* 您只能在root之后指定一个级别的谓词。它们将被递归地遍历。标量和实体节点的处理是相似的。
* 建议每个查询只使用一个递归块。
* 要小心，因为结果的大小可能会很快爆炸，如果结果集变得太大，就会返回一个错误。在这种情况下，使用更多的过滤器，使用分页限制结果，或在根处提供深度参数，如上面的示例所示。
* `loop`参数可以设置为`false`，在这种情况下，导致循环的路径将在遍历时被忽略。
* 如果未指定，则`loop`参数的值默认为`false`。
* 如果`loop`参数的值为`false`并且没有指定`depth`, `depth`将默认为`math`。maxxuint64，这意味着可能遍历整个图，直到到达所有叶节点。