# 外部ID和插入块 (External IDs and Upsert Block)

`upsert`块使得管理外部`id`很容易。

首先，设置如下`Schema`：

``` schema
xid: string @index(exact) .
<http://schema.org/name>: string @index(exact) .
<http://schema.org/type>: [uid] @reverse .
```

然后设置类型：

``` dql
{
  set {
    _:blank <xid> "http://schema.org/Person" .
    _:blank <dgraph.type> "ExternalType" .
  }
}
```

现在您可以创建一个新的`person`并使用`upsert`块添加其类型：

``` dql 
upsert {
    query {
      var(func: eq(xid, "http://schema.org/Person")) {
        Type as uid
      }
      var(func: eq(<http://schema.org/name>, "Robin Wright")) {
        Person as uid
      }
    }
    mutation {
        set {
          uid(Person) <xid> "https://www.themoviedb.org/person/32-robin-wright" .
          uid(Person) <http://schema.org/type> uid(Type) .
          uid(Person) <http://schema.org/name> "Robin Wright" .
          uid(Person) <dgraph.type> "Person" .
        }
    }
  }
```

还可以删除`person`并分离`Type`和`person`节点之间的关系。这和上面是一样的，但是你使用了关键字`delete`而不是`set`。`http://schema.org/Person`将保留，但`Robin Wright`将被删除：

``` dql
upsert {
    query {
      var(func: eq(xid, "http://schema.org/Person")) {
        Type as uid
      }
      var(func: eq(<http://schema.org/name>, "Robin Wright")) {
        Person as uid
      }
    }
    mutation {
        delete {
          uid(Person) <xid> "https://www.themoviedb.org/person/32-robin-wright" .
          uid(Person) <http://schema.org/type> uid(Type) .
          uid(Person) <http://schema.org/name> "Robin Wright" .
          uid(Person) <dgraph.type> "Person" .
        }
    }
  }
```

再次查询一个用户：

``` dql
{
  q(func: eq(<http://schema.org/name>, "Robin Wright")) {
    uid
    xid
    <http://schema.org/name>
    <http://schema.org/type> {
      uid
      xid
    }
  }
}
```