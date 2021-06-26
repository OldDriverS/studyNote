# 表的配置



表是nftables规则集中的顶级容器;它们包含链(chain)、集合（set）、maps、 [流表（flowtables）](https://wiki.nftables.org/wiki-nftables/index.php/Flowtables),和[有状态对象（stateful objects）](https://wiki.nftables.org/wiki-nftables/index.php/Stateful_objects)。

每个表恰好属于一个协议簇（family）。因此，您的规则集要求对于要过滤的每个簇至少有一个表。

表配置的一些基本操作和命令如下:

## 添加一个表

```bash
% nft add table ip filter
```

## 列出已有的表

```bash
% nft list tables
```

## 删除一个表（以foo表为例）

```
% nft delete table ip foo
```

## 清除指定表中的规则（以filter表为例）

可以使用如下命令删除属于该表的所有规则：

```
% nft flush table ip filter
```

他删除了表中每个链的规则。

注意:*nft flush table ip filter* 不会刷新该表中定义的**Sets**，如果要刷新的表不存在且您使用的是Linux <4.9.0，则会导致错误，这可以通过刷新规则集来克服。