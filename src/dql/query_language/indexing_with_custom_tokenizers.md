# 使用自定义 `tokenizer` 进行索引

`Dgraph`自带一个内置索引的大工具箱，但有时对于小众用例来说，它们总是不够用。

为了填补空白，`Dgraph`允许你通过插件系统实现定制的标记器。

## Caveats

插件系统使用`Go`的[pkg/plugin](https://golang.org/pkg/plugin/)。这给插件的使用带来了一些限制。

* 插件必须用`Go`编写。
* 从1.9开始，`pkg/plugin`只能在`Linux`上工作。因此，插件只能在部署在`Linux`环境中的`Dgraph`实例上工作。
* 用于编译插件的`Go`版本应该与用于编译`Dgraph`本身的`Go`版本相同。`Dgraph`总是使用最新版本的`Go`(你也应该如此!)

### 实现一个插件

