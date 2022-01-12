# 使用cURL提交变更 (Mutations Using cURL)

变更可以通过向`Alpha`的`/mutate`发出`POST`请求来实现。在命令行上，这可以通过`curl`完成。在`URL`中传递参数`commitNow=true`表明你是要提交变更。

提交一个`set`变更：

``` bash 
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
{
  set {
    _:alice <name> "Alice" .
    _:alice <dgraph.type> "Person" .
  }
}'
```

提交一个`delete`变更：

``` bash 
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
{
  delete {
    # Example: The UID of Alice is 0x56f33
    <0x56f33> <name> * .
  }
}'
```

提交一个存储在文件中的变更，使用`--data-binary`选项将不会对文件数据编码：

``` bash 
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true --data-binary @mutation.txt
```
