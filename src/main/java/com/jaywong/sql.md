

## SQL优化 ##
    1、SQL中过滤条件放在where/on后面的区别:
        a. inner join: 两者没有区别
        b. left join : left join的特殊机制就是on后面的条件只对右表起作用, 所以即使在on后面对左表进行过滤, 结果左表的数据还是会存在, 但右表相应的数据会被过滤掉
        c. right join: 同left join
    2、避免select * 查询, 改用具体字段, 节省资源, 减少网络开销
    3、尽量避免使用or改用union all同表: or可能会使索引失去作用, 比如第一个字段使用了索引, 但or后的字段是非索引字段, 会进行全表扫描, 最后合并
        也就是整体分成全表扫描 + 索引扫描 + 合并。 如果使用union all只扫一次全表扫描即可
    4、尽量使用数值型替代字符型: 引擎在处理数据时会逐个字符进行比较, 并且增加存储开销
    5、使用varchar替代char: varchar按照实际长度进行存储, 存储空间小, 节省空间
        char按照声明大小存储, 不足补空
        varchar的查询效率相较于char来说会更高
    6、避免在where子句中使用 != 或 <> 操作符: 使用该字符会使索引失效从而进行全表扫描
    7、inner left right 三种join优先使用inner: 优化原则, 小表驱动大表
    8、提高group by效率: 先过滤，后分组
    9、清空表时优先使用truncate: truncate 比 delete使用的资源和事务日志资源少, delete按行删除, 每次都会留下一次事务日志, 但truncate是释放存储表数据所用的数据页来进行删除
        truncate删除表中所有行值, 新行标识所用的计数值重置为该列的种子
        delete会保留标识计数值
        有外键的表不能使用truncate只能用delete
    10、操作delete或者update语句的时候, 可以加个limit或者循环批次删除:
        a. 降低误删除代价
        b. 避免长事务, 执行delete时, 会将所有相关行加锁, 所有执行相关操作都会被锁住, 若数据大, 则会导致数据库长时间无法操作
        c. 一次性删除太多数据, 会造成锁表, lock wait timeout错误
    11、UNION操作: 尽量使用union all代替union操作(若无重复数据): union会对结果数据进行排序后删除重复项, 资源开销大, union all不会对数据去重, 只是做合并操作
    12、避免在索引列上使用内置函数: 会导致索引失效
    13、复合索引的最左特性: 索引是两列(id, name):
        a. 查询只出现最左索引(id), 索引生效
        b. 查询只出现非最左索引(name), 索引失效
        c. 复合索引全部使用, 索引生效
    14、like模糊匹配优化: 可能会使索引失效
        a. 尽量使用右模糊查询, like '...%'
        b. 左模糊无法使用索引, 但可利用reverse
        c. like全模糊会使索引失效


## hive优化 ##
    执行顺序:
    第一步：确定数据源 FROM JOIN ON
    第二步：过滤数据 WHERE GROUP BY (开始使用SELECT 中的别名，后面的语句中都可以使用) 内置函数(avg，sum...) HAVING
    第三步：查询数据 SELECT
    第四步：显示数据 DISTINCT ORDER BY LIMIT
    1、使用分区, 分桶
    2、使用sort by 替代order by: order by会进行全局排序, 这会导致所有map端数据都进入一个reduce中，在数据量大时可能会长时间计算不完。
        如果使用sort by，那么就会视情况启动多个reducer进行排序，并且保证每个reducer内局部有序。为了控制map
        端数据分配到reduce的key，往往还要配合distribute by一同使用。如果不加distribute by的话，map端数据
        就会随机分配给reducer。
    3、使用group by替代distinct, 使用group by后再count替代count(distinct)
    4、聚合技巧: grouping sets 、cube、rollup            
        a. grouping sets: 多列的group by count(), 可以用一个grouping sets实现
        b. cube: 根据group by维度的所有组合进行聚合
        c. rollup: 以最左侧的维度为主，进行层级聚合，是cube的子集           
    5、union all时可以开启并发执行: Hive中互相没有依赖关系的job间是可以并行执行的
        set hive.exec.parallel=true;
    6、小表join大表:
        MapJoin:  MapJoin 会把小表全部读入内存中，在map阶段直接拿另外一个表的数据和内存中表数据做匹配，由于在map是进行了join操作，省去了reduce 阶段，
        运行的效率就会高很多。select 时加上/*+mapjoin(small_table)*标注。小表标准: 数据不超过2万条, 大小不超过25M为宜
    7、大表join大表:
        a. 分桶: 可以考虑使用分桶表 SMB(Sort Merge Bucket)
            大表与大表join时,如果key分布均匀,单纯因为数据量过大,导致任务失败或运行时间过长, 可以考虑将大表分桶,来优化任务
            原理 : key % 分桶数 = 分桶编号    分桶编号1 join 分桶编号1
            注意 : A表、B表 都需要是分桶表且分桶规则相同  创建分桶表 clustered by() sorted by (id) into 2 buckets
        b. mapjoin: 表是否可以先过滤掉无用数据, 过滤后看是否能够满足mapjoin("/*+mapjoin(b)*/")的操作数量
        c. join时采用case when: 若哪个key产生倾斜很明确且数量少, 就可以将这些值随机的分发到Reduce, 逻辑在于join时对这些特殊值进行concat随机数,
            已达到随机分发的目的. PS: hive已优化, 通过skewinfo和skewjoin参数设置
    8、设置自动选择MapJoin:
        a. set hive.auto.convert.join = true; 默认为 true;
        b. set hive.mapjoin.smalltable.filesize=25000000; 25M以下为小表

## hive在join过程中会出现很多小文件, 如何优化: ##
    

## MR的shuffle和Spark的shuffle区别: ##



## 窗口函数: ##
    1、over()
        定分析函数工作的数据窗口大小, 这个数据窗口大小可能会随着行的变化而变化
    2、current row（当前行）
        n preceding: 往前n行
        n following: 往后n行
    3、unbounded（无边界）
        unbounded preceding: 往前无边界
        unbounded following: 往后无边界
    4、lag()
        往前第n行
    5、lead()
        往后第n行

## 4个By区别: ##
    order by: 全局排序, 只有一个reducer, 在生产环境中Order By用的比较少，容易导致OOM
    sort by : 分区内排序
    distribute by : 类似MR中的Partition, 进行分区, 结合sort by使用, 在生产环境中Sort By+ Distrbute By用的多
    cluster by: 当Distribute by和Sorts by字段相同时，可以使用Cluster by方式。Cluster by除了具有Distribute by的
    功能外还兼具Sort by的功能。但是排序只能是升序排序，不能指定排序规则为ASC或者DESC

## row_number, dense_rank, rank 区别: ##
    a. row_number: 不考虑重复值, 顺序排序
    b. dense_rank: 考虑重复值, 当值相同时, 排名相同, 下一值会接着顺序排序
    c. rank: 考虑重复值, 当值相同时, 排名相同, 下一值的顺序会跳值



