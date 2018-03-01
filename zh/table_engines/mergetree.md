<a name="table_engines-mergetree"></a>

# MergeTree

`MergeTree` 允许您依据主键和日期创建索引，并进行实时的数据更新操作。`MergeTree` 是 ClickHouse 里最为先进的表引擎。请注意不要将 `MergeTree` 跟 [Merge](merge.md) 引擎混淆。

`MergeTree` 引擎在创建时接收以下4个参数，

- 日期字段的名称 （索引字段）
- 采样表达式 （可选的）
- 含有主键相关字段的元组
- 稀疏索引的粒度（见下文）。示例：

不使用采样表达式的例子：

```sql
MergeTree(EventDate, (CounterID, EventDate), 8192)
```

使用采样表达式的例子：

```sql
MergeTree(EventDate, intHash32(UserID), (CounterID, EventDate, intHash32(UserID)), 8192)
```

以 MergeTree 作为引擎的数据表必须含有一个独立的 `Date` 字段。比如说， `EventDate` 字段。这个日期字段必须是 `Date` 类型的（非 `DateTime` 类型）。

主键可以是任意表达式构成的元组（通常是列名称的元组），或者是单独一个字段。

抽样表达式（可选的）可以是任意表达式。这个表达式必须在主键中。上面的例子使用了 `CounterID` 的哈希 `intHash32` 作为采样表达式，旨在近乎随机地在 `CounterID` 和 `EventDate` 内打乱数据条目。换而言之，当我们在查询中使用 `SAMPLE` 子句时，我们就可以得到一个近乎随机分布的用户列表。

数据表将数据分割为小的索引块作为单位进行处理。 每个索引块之间依照主键排序。每个索引块记录了指定的开始日期和结束日期。在您插入数据时，MergeTree 就会对数据进行排序处理，以保证存储在索引块内的数据有序。 索引块之间的合并过程会在系统后台定期自动执行。MergeTree 引擎会选择几个相邻的索引块进行合并（通常是较小的索引块）， 然后对二者合并、排序。

具体而言, 向 MergeTree 表中插入数据时，引擎会首先对新数据执行递增排序而保存索引块；其后，数据索引块之间又会进一步合并，以减少总体索引块数量。 因此，合并过程本身并无过多排序工作。

向 MergeTree 插入数据时，不同月份的数据会被自动分散在不同索引块中。不同月份的索引块不会被合并。这是为了便于本地化数据修改（以及备份）。

索引块合并时设有体积上限，以避免索引块合并产生庞大的新索引块。

除了保存索引块中的数据外, 引擎会额外保存一个索引文件，以储存每 `index_granularity` 行的主键值和对应位置，这就构成了对有序数据的稀疏的索引。

对列而言，MergeTree 在每`index_granularity`行的位置也写入了标记，从而确定数据所在的范围，以便查找。

当使用 `SELECT` 读取表内数据时， MergeTree 会判断是否能够使用索引。以下两种情况里，索引将被使用：
1. 当 `WHERE` 语句或 `PREWHERE` 语句用于判断相等或不等判关系时 （作为子句）；
2. 或当 `IN` 语句的对象为主键或者 `Date` 或者 它们之间的逻辑关系。

因此， MergeTree 能够快速查询一个或多个主键范围的值。在下面的示例中，MergeTree 能够快速的查询一个明确的 `CounterID` ，指定范围的日期区间里的一个明确的 `CounterID` ，各种 `CounterID` 的集合等。

```sql
SELECT count() FROM table WHERE EventDate = toDate(now()) AND CounterID = 34
SELECT count() FROM table WHERE EventDate = toDate(now()) AND (CounterID = 34 OR CounterID = 42)
SELECT count() FROM table WHERE ((EventDate >= toDate('2014-01-01') AND EventDate <= toDate('2014-01-31')) OR EventDate = toDate('2014-05-01')) AND CounterID IN (101500, 731962, 160656) AND (CounterID = 101500 OR EventDate != toDate('2014-05-01'))
```

上面例子中的查询会使用基于日期和主键的索引。索引也可以被用在更加复杂的查询之中；读取表的过程是按部就班进行的，所以使用索引绝不会比全表搜索耗时。

示例贴：

可以看到，下面的例子中， MergeTree 无法使用索引。

```sql
SELECT count() FROM table WHERE CounterID = 34 OR URL LIKE '%upyachka%'
```

若要知晓 MergeTree 能否在查询中使用索引, 请配置系统参数 [force_index_by_date](../operations/settings/settings.md#settings-settings-force_index_by_date)  、 [force_primary_key](../operations/settings/settings.md#settings-settings-force_primary_key).

全局的索引之中仅仅保存了单个数据索引块的日期范围。然而，一个数据索引块可能包含很多日期的数据（甚乎整月），MergeTree 在数据索引块内部依照主键排序，然而用于分组的日期并不一定在数据表的首列。因此，在查询语句中，如果只有日期范围而没有限定主键范围，这将可能导致不必要的数据读取。

对于并发查询， MergeTree 使用了多版本管理 ： 当我们试图同时读取、写入数据时，查询操作将会在已经插入完毕的索引快中进行，而排除没有写入完毕的索引块，正在被写入的块因而不会受到干扰，这个过程没有使用任何锁机制，同时插入操作不会阻塞读取操作。

对 MergeTree 进行读取的操作会在引擎内部自动的被并行执行。

MergeTree 支持 `OPTIMIZE` 语句，它会调用额外的合并步骤。

它可以管理一张很大的数据表，我们也可以小批量、连续地向其添加数据，这正是 MergeTree 设计之初衷。

它支持数据备份功能，具体见 [数据副本](replication.md) 一章。
