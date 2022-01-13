# gRPC 客户端

## 支持的`Dgraph`版本

## 快速开始

### 小例子

## 使用客户端

### 创建一个实例

### 多用户

### 为`Dgraph Cloud`创建客户端实例

### 修改数据库

### 创建一个事务

### 执行`mutation`

### 执行`query`

### 执行一个`Upsert` (`Query` + `Mutation`)

`txn.doRequest`函数允许你提交由一个`mutation`和`query`的`upsert`。`query`块中定义的变量可以在`mutation`块中使用。你当然也能使用`txn.doRequest`但是只设置一个`query`或者`mutation`。

想知道更多关于`upsert`块的信息，浏览[upsert]

``` js
const query = `
  query {
      user as var(func: eq(email, "wrong_email@dgraph.io"))
  }`

const mu = new dgraph.Mutation();
mu.setSetNquads(`uid(user) <email> "correct_email@dgraph.io" .`);

const req = new dgraph.Request();
req.setQuery(query);
req.setMutationsList([mu]);
req.setCommitNow(true);

// Upsert: If wrong_email found, update the existing data
// or else perform a new mutation.
await dgraphClient.newTxn().doRequest(req);
```


### 执行一个条件插入 (`conditional upsert`)

条件插入块允许你通过`@if`指令指定一个条件来执行`mutation`块。

``` js
const query = `
  query {
      user as var(func: eq(email, "wrong_email@dgraph.io"))
  }`

const mu = new dgraph.Mutation();
mu.setSetNquads(`uid(user) <email> "correct_email@dgraph.io" .`);
mu.setCond(`@if(eq(len(user), 1))`);

const req = new dgraph.Request();
req.setQuery(query);
req.addMutations(mu);
req.setCommitNow(true);

await dgraphClient.newTxn().doRequest(req);
```

### 提交一个事务

*todo*

``` js
const txn = dgraphClient.newTxn();
try {
  // ...
  // Perform any number of queries and mutations
  // ...
  // and finally...
  await txn.commit();
} catch (e) {
  if (e === dgraph.ERR_ABORTED) {
    // Retry or handle exception.
  } else {
    throw e;
  }
} finally {
  // Clean up. Calling this after txn.commit() is a no-op
  // and hence safe.
  await txn.discard();
}
```

### 释放资源

必须显式地调用`DgraphClientStub`示例的`DgraphClientStub#close()`方法释放`gRPC`客户端占用的资源。

``` js
const SERVER_ADDR = "localhost:9080";
const SERVER_CREDENTIALS = grpc.credentials.createInsecure();

// Create instances of DgraphClientStub.
const stub1 = new dgraph.DgraphClientStub(SERVER_ADDR, SERVER_CREDENTIALS);
const stub2 = new dgraph.DgraphClientStub(SERVER_ADDR, SERVER_CREDENTIALS);

// Create an instance of DgraphClient.
const dgraphClient = new dgraph.DgraphClient(stub1, stub2);

// ...
// Use dgraphClient
// ...

// Cleanup resources by closing all client stubs.
stub1.close();
stub2.close();
```

### 调试模式

``` js
// Create a client.
const dgraphClient = new dgraph.DgraphClient(...);

// Enable debug mode.
dgraphClient.setDebugMode(true);
// OR simply dgraphClient.setDebugMode();

// Disable debug mode.
dgraphClient.setDebugMode(false);
```

### 设置请求的元信息

``` js
// The following pi ece of code shows how one can set metadata with
// auth-token, to allow Alter operation, if the server requires it.

var meta = new grpc.Metadata();
meta.add('auth-token', 'mySuperSecret');

await dgraphClient.alter(op, meta);
```