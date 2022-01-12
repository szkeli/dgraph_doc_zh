# 检索调试信息

每个`Dgraph`数据节点都在`/debug/pprof`端点上公开概要文件，在`/debug/vars`端点上公开度量。每个Dgraph数据节点都有自己的分析和度量信息。下面是Dgraph公开的调试信息列表，以及检索这些信息的相应命令。

## Metrics 信息

如果想从`Dgraph`实例外部收集这些`Metrics`信息，则需要在`Alpha`启动时传递`--expose_trace=true`标志，否则仅可以通过`localhost`收集`metrics`信息。

``` bash
curl http://<IP>:<HTTP_PORT>/debug/vars
```

`Metrics`信息也能在`/debug/promethus_metrics`通过`Promethus`的格式检索。详情查看[链接](https://dgraph.io/docs/deploy/metrics/)

## 性能信息

性能信息能够通过`GO`内建的工具`go tool pprof`查看，可以浏览[这篇帖子](https://blog.golang.org/profiling-go-programs)关于`pprof`工具的使用。

每一个`Dgraph Alpha`和`Dgraph Zero`都会通过`HTTP`端口在`/debug/pprof/<profile>`暴露性能信息：

``` bash
go tool pprof http://<IP>:<HTTP_PORT>/debug/pprof/heap
Fetching profile from ...
Saved Profile in ...
```

上面的命令运行成功之后将输出性能信息的保存位置。

在交互式的pprof shell中，你可以使用像top这样的命令来获取配置文件中top函数的列表，web命令来获取在web浏览器中打开的配置文件的可视化图形，或者list命令来显示覆盖了配置信息的代码列表。

## CPU Profile

``` bash
go tool pprof http://<IP>:<HTTP_PORT>/debug/pprof/profile
```

## 内存 Profile

``` bash
go tool pprof http://<IP>:<HTTP_PORT>/debug/pprof/heap
```

## 块 Profile

默认情况下，Dgraph不收集块配置文件。Dgraph必须以——profile_mode=block和——block_rate=<n>和N> 1开始。</n>

``` bash
go tool pprof http://<IP>:<HTTP_PORT>/debug/pprof/block
```

