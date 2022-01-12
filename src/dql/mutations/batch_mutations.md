# 批量变更 (Batch Mutations)

每个变更可以包含多个`RDF`三元组。对于大量的数据上传，许多这样的变更可以并行批处理。命令`dgraph live`就是这样做的；默认情况下，将1000个`RDF`行批处理到一个查询中，同时并行运行100个这样的查询。

`dgraph live`以`gzip`的`N-Quad`文件(即不带`{set{`的三元组列表)作为输入，并为输入中的所有三元组批量处理变更。使用如下命令查看该工具的文档：

``` bash 
dgraph live --help
```

> 更多内容查看[快速数据导入](https://dgraph.io/docs/deploy/fast-data-loading/overview/)