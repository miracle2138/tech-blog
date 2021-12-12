# count(*)

## 执行过程
如果count语句包括where，肯定需要扫描记录才能知道。
如果不带where，在innodb中，由于支持mvcc，所以还是要扫描记录，根据undo log判断要不要计数。
所以在innodb中，count语句肯定要扫描数据行，而不是直接返回。

## count(*)
对于count(字段)，需要遍历数据行取出列值判断是不是为空，如果不为空计数+1。
对于count(*)，mysql有优化，只遍历计数不取值，所以效率更高，和count(1)差不多。
统计行数，直接用count(*)即可。

## 参考
- [14 | count(*)这么慢，我该怎么办？](https://time.geekbang.org/column/article/72775)