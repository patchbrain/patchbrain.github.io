---
layout: default
title:  "对Postgres Explain文档的解释"
date:   2024-09-06 10:40:00 +0800
categories: Postgres
excerpt: "Postgres的执行计划可以帮助我们查看SQL背后的执行原理，更容易定位SQL中的性能优化点"
---
参考文档链接: https://www.postgresql.org/docs/current/using-explain.html

以该执行计划的部分文本为例： `Seq Scan on tenk1  (cost=0.00..458.00 rows=10000 width=244)`，对应SQL:`EXPLAIN SELECT * FROM tenk1;`

从左到右：

`0.00`：启动成本，扫描第一行数据之前的成本

`458.00`: 检索所有行的成本

`10000`: 该node输出的行数

`244`: 输出的平均每行大小(bytes)


成本单位的划定：通常以在磁盘中获取页面作为成本单位，也就是`seq_page_cost=1`，其他操作的成本根据`seq_page_cost`来设定


父节点的成本应该包含所有子节点成本。另外`rows`这个字段并不代表扫描的行数，而是该`node`发往上层进行处理的行数。

<h2 id="aTAUS">加上Where条件</h2>
SQL为`EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 7000;`

Explain结果为：

```plain
Seq Scan on tenk1  (cost=0.00..483.00 rows=7001 width=244)
   Filter: (unique1 < 7000)
```



该node会对扫描到的每一行都用`Filter`进行筛选，对每行的筛选需要CPU成本，这部分成本为：`10000*cpu_operator_cost`

<h2 id="APD7o">继续限制行数，使用到了索引</h2>
SQL:`EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100;`

Explain:

```plain
Bitmap Heap Scan on tenk1  (cost=5.07..229.20 rows=101 width=244)
   Recheck Cond: (unique1 < 100)
   ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=101 width=0)
         Index Cond: (unique1 < 100)
```



底层node为`Bitmap Index Scan`， 代表扫描了索引`tenk1_unique1`

扫描完毕后，由于`select`项为全表，因此还要去磁盘中拿其他列的数据。

由于随机访问磁盘的成本较高，因此在底层节点的结果基础上按物理存储顺序排序，再进行扫描。

因为顺序扫描的成本较低。



`Recheck`的原因：因为`Bitmap Index Scan`只是记录了满足条件的行到位图中，如果这期间这些行位置所对应的行数据有变动，不使用`Recheck`就可能会造成结果错误。

<h2 id="CSrCL">多个排序字段和LIMIT，触发增量排序</h2>
SQL: `EXPLAIN SELECT * FROM tenk1 ORDER BY four, ten LIMIT 100;`

Explain: 

```plain
Limit  (cost=521.06..538.05 rows=100 width=244)
   ->  Incremental Sort  (cost=521.06..2220.95 rows=10000 width=244)
         Sort Key: four, ten
         Presorted Key: four
         ->  Index Scan using index_tenk1_on_four on tenk1  (cost=0.29..1510.08 rows=10000 width=244)
```

首先，按four排序，直接进行了索引扫描`Index Scan using index_tenk1_on_four`

由于使用了`Limit`，PG对这部分结果进行增量排序`Incremental Sort`，不必排序完毕才返回结果。并且增量排序是分批的，避免了对内存的过多使用造成外溢到磁盘中进行排序。

<h2 id="WEcAR">使用多个有索引的筛选字段</h2>
此处进行两个SQL的执行计划对比，看看在加上`Limit`后会出现什么变化，并分析原因。



SQL1: `EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000;`

Explain: 

```plain
 Bitmap Heap Scan on tenk1  (cost=25.08..60.21 rows=10 width=244)
   Recheck Cond: ((unique1 < 100) AND (unique2 > 9000))
   ->  BitmapAnd  (cost=25.08..25.08 rows=10 width=0)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=101 width=0)
               Index Cond: (unique1 < 100)
         ->  Bitmap Index Scan on tenk1_unique2  (cost=0.00..19.78 rows=999 width=0)
               Index Cond: (unique2 > 9000)
```

执行计划表明，针对两个索引分别进行了一次`Bitmap Index Scan`，并将两个扫描的结果进行与操作，得到结果集。

在上层节点中，把该结果集进行`Recheck`并且取到其他字段的数据。



SQL2: `EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000 LIMIT 2;`

Explain:

```plain
 Limit  (cost=0.29..14.48 rows=2 width=244)
   ->  Index Scan using tenk1_unique2 on tenk1  (cost=0.29..71.27 rows=10 width=244)
         Index Cond: (unique2 > 9000)
         Filter: (unique1 < 100)
```

下层`Index Scan`节点，首先使用`unique2>9000`扫描索引获得结果集，再用`unique1<100`对结果集的每行数据进行筛选，进一步得到结果集。预估会扫描得到10行结果，但是上层`LIMIT`节点限制了2条结果，因此当获取两条数据时立即结束，2条结果和10条结果相差5倍，因此上层节点的总成本约为：`71.27/(10/2)≈14.25`



除了`LIMIT`的原因，构建`Bitmap`是需要成本的，因此启动成本相较于`Index Scan`更高，所以最后使用`Index Scan`

<h2 id="hopih">联表</h2>
SQL: `Explain select * from tenk1 t1, tenk2 t2 where t1.unique1 < 10 and t1.unique2 = t2.unique2`

Explain:

```plain
 Nested Loop  (cost=4.65..118.62 rows=10 width=488)
   ->  Bitmap Heap Scan on tenk1 t1  (cost=4.36..39.47 rows=10 width=244)
         Recheck Cond: (unique1 < 10)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..4.36 rows=10 width=0)
               Index Cond: (unique1 < 10)
   ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..7.91 rows=1 width=244)
         Index Cond: (unique2 = t1.unique2)
```

首先使用`Bitmap Index Scan + Bitmap Heap Scan`取出了tenk1中满足`unique1<10`的所有行，共有10行，为`join`的外部表。

下一个节点`Index Scan`则是使用索引`tenk2_unique2`通过条件`unique2=t1.unique2`在上一个节点返回的10行中筛选出结果，只有一行，为内部表。

由于内外表较少，使用`Nested Loop`可以很快给出结果。成本为`7.91*10 + 一些Join产生的CPU成本 ≈ 118.62`

<h2 id="IHyh1">连接后进一步筛选</h2>
Join后的行数有可能不是`扫描节点A行数*扫描节点B行数`，而是可能更小，比如在连接后使用条件过滤。

SQL: 

```plain
EXPLAIN SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t2.unique2 < 10 AND t1.hundred < t2.hundred;
```

Explain:

```plain
Nested Loop  (cost=4.65..49.46 rows=33 width=488)
   Join Filter: (t1.hundred < t2.hundred)
   ->  Bitmap Heap Scan on tenk1 t1  (cost=4.36..39.47 rows=10 width=244)
         Recheck Cond: (unique1 < 10)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..4.36 rows=10 width=0)
               Index Cond: (unique1 < 10)
   ->  Materialize  (cost=0.29..8.51 rows=10 width=244)
         ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..8.46 rows=10 width=244)
               Index Cond: (unique2 < 10)
```

用筛选条件`unique1<10`对t1进行`Bitmap Index Scan`，获得10行数据作为外部表。

用筛选条件`unique2<10`对t2进行`Index Scan`，获得10行数据作为内部表。并把数据物化`Materialize`，也就是放在内存中，从而不用每次都再去获取t2数据。

`t1.hundred<t2.hundred`依附于`Join Filter`，这是连接条件，只有符合该条件的会被连接。最后结果只有33条数据，小于`10*10=100`条。 但这并不改变两个扫描节点所得到的行数。

<h2 id="WaIDH">使用Hash Join</h2>
当两个节点扫描结果差异较大时，有可能会使用`HashJoin`，会把较小的那个结果集构建为哈希表，然后顺序扫描大结果集，扫描的每行都会在哈希表中比较。

SQL: 

```plain
EXPLAIN SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2;
```

Explain:

```plain
 Hash Join  (cost=230.47..713.98 rows=101 width=488)
   Hash Cond: (t2.unique2 = t1.unique2)
   ->  Seq Scan on tenk2 t2  (cost=0.00..445.00 rows=10000 width=244)
   ->  Hash  (cost=229.20..229.20 rows=101 width=244)
         ->  Bitmap Heap Scan on tenk1 t1  (cost=5.07..229.20 rows=101 width=244)
               Recheck Cond: (unique1 < 100)
               ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=101 width=0)
                     Index Cond: (unique1 < 100)
```

在t1上进行`Bitmap Heap Scan`的结果被作为构建哈希表的输入数据。

并且顺序扫描t2，将每行数据在哈希表中取值看是否匹配。

<h2 id="xTBpA">使用merge join</h2>
`merge join`需要两个节点的数据都处于排序状态，排序字段为连接条件的字段。

SQL: 

```plain
EXPLAIN SELECT *
FROM tenk1 t1, onek t2
WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2;
```

Explain:

```plain
 Merge Join  (cost=198.11..268.19 rows=10 width=488)
   Merge Cond: (t1.unique2 = t2.unique2)
   ->  Index Scan using tenk1_unique2 on tenk1 t1  (cost=0.29..656.28 rows=101 width=244)
         Filter: (unique1 < 100)
   ->  Sort  (cost=197.83..200.33 rows=1000 width=244)
         Sort Key: t2.unique2
         ->  Seq Scan on onek t2  (cost=0.00..148.00 rows=1000 width=244)
```

由于t1在unique1上有索引，因此直接使用索引扫描，得到的结果集按unique1排序。

t2由于没有类似于`unique2<100`的条件限制，因此取出的行会很多。这种情况下使用索引扫描会造成大量的随机IO，在性能上不如`顺序扫描+排序`，因此对t2进行顺序扫描。

最后，将两个节点的结果通过`Merge Join`连接。

<h1 id="zqdsg">Explain Analyze</h1>
相比单独的`Explain`能展示更多精确的信息，比如真实的扫描行数，真实的启动/执行时间等。

以下面SQl及其执行计划为例：

SQL: 

```plain
EXPLAIN ANALYZE SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t1.unique2 = t2.unique2;
```

Explain Analyse:

```plain
Nested Loop  (cost=4.65..118.62 rows=10 width=488) (actual time=0.128..0.377 rows=10 loops=1)
   ->  Bitmap Heap Scan on tenk1 t1  (cost=4.36..39.47 rows=10 width=244) (actual time=0.057..0.121 rows=10 loops=1)
         Recheck Cond: (unique1 < 10)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..4.36 rows=10 width=0) (actual time=0.024..0.024 rows=10 loops=1)
               Index Cond: (unique1 < 10)
   ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..7.91 rows=1 width=244) (actual time=0.021..0.022 rows=1 loops=10)
         Index Cond: (unique2 = t1.unique2)
 Planning time: 0.181 ms
 Execution time: 0.501 ms
```

其中`actual time`以毫秒为单位，并且只是在理论上和真实值接近。

`loops`代表执行该节点的次数，`actual time`是每次执行节点时的平均时长。



如果执行计划中有`Sort`和`Hash`，则会展示更多信息，比如：

```plain
 Sort  (cost=717.34..717.59 rows=101 width=488) (actual time=7.761..7.774 rows=100 loops=1)
   Sort Key: t1.fivethous
   Sort Method: quicksort  Memory: 77kB
   ->  Hash Join  (cost=230.47..713.98 rows=101 width=488) (actual time=0.711..7.427 rows=100 loops=1)
         Hash Cond: (t2.unique2 = t1.unique2)
         ->  Seq Scan on tenk2 t2  (cost=0.00..445.00 rows=10000 width=244) (actual time=0.007..2.583 rows=10000 loops=1)
         ->  Hash  (cost=229.20..229.20 rows=101 width=244) (actual time=0.659..0.659 rows=100 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 28kB
               ->  Bitmap Heap Scan on tenk1 t1  (cost=5.07..229.20 rows=101 width=244) (actual time=0.080..0.526 rows=100 loops=1)
                     Recheck Cond: (unique1 < 100)
                     ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=101 width=0) (actual time=0.049..0.049 rows=100 loops=1)
                           Index Cond: (unique1 < 100)
 Planning time: 0.194 ms
 Execution time: 8.008 ms
```

Sort节点展示了`Sort Method: quicksort  Memory: 77kB`，可以看到使用的是快速排序，并且使用内存为`77kB`

Hash节点展示了`Buckets: 1024  Batches: 1  Memory Usage: 28kB`，代表使用了1024个桶，并在一个批次内将数据加载到哈希表中，使用内存为`28kB`



也会显示被筛选条件排除的行数，比如：

```plain
EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE ten < 7;

                                               QUERY PLAN
-------------------------------------------------------------------​--------------------------------------
 Seq Scan on tenk1  (cost=0.00..483.00 rows=7000 width=244) (actual time=0.016..5.107 rows=7000 loops=1)
   Filter: (ten < 7)
   Rows Removed by Filter: 3000
 Planning time: 0.083 ms
 Execution time: 5.905 ms
```

`Rows Removed by Filter: 3000`代表有3000行不符合筛选条件



两个时间：

`Planning time`指的是计划器生成计划树的实际，而不包含解析/重写SQL的时间。

`Executing time`指的是执行器执行的时间，不包含解析/重写SQL时间及计划时间。



<h2 id="fuNfd">关闭顺序扫描</h2>
SQL:

```plain
SET enable_seqscan TO off;

EXPLAIN ANALYZE SELECT * FROM polygon_tbl WHERE f1 @> polygon '(0.5,2.0)';
```

Explain:

```plain
Index Scan using gpolygonind on polygon_tbl  (cost=0.13..8.15 rows=1 width=32) (actual time=0.062..0.062 rows=0 loops=1)
   Index Cond: (f1 @> '((0.5,2))'::polygon)
   Rows Removed by Index Recheck: 1
 Planning time: 0.034 ms
 Execution time: 0.144 ms
```

由于表的行数太少，如果不执行`SET enable_seqscan TO off;`，计划器会选择顺序扫描而不是索引扫描。

此处由于用索引扫描无法精确判断哪些`f1`能够包含`'((0.5,2))'::polygon`，因此会进行`recheck`。

`Rows Removed by Index Recheck: 1`代表在`Recheck`过程中筛选排除了1行，因此可以看到`rows=1`但是`actual rows=0`



<h2 id="eazSS">显示Buffers信息</h2>
SQL: `EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000;`

Explain:

```plain
Bitmap Heap Scan on tenk1  (cost=25.08..60.21 rows=10 width=244) (actual time=0.323..0.342 rows=10 loops=1)
   Recheck Cond: ((unique1 < 100) AND (unique2 > 9000))
   Buffers: shared hit=15
   ->  BitmapAnd  (cost=25.08..25.08 rows=10 width=0) (actual time=0.309..0.309 rows=0 loops=1)
         Buffers: shared hit=7
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=101 width=0) (actual time=0.043..0.043 rows=100 loops=1)
               Index Cond: (unique1 < 100)
               Buffers: shared hit=2
         ->  Bitmap Index Scan on tenk1_unique2  (cost=0.00..19.78 rows=999 width=0) (actual time=0.227..0.227 rows=999 loops=1)
               Index Cond: (unique2 > 9000)
               Buffers: shared hit=5
 Planning time: 0.088 ms
 Execution time: 0.423 ms
```

`Buffers: shared hit`代表节点命中了多少次共享内存区，单位为`数据页`。

<h2 id="Dl3uA">Trigger</h2>
`BEFORE TRIGGER`会在执行计划中体现，但是`AFTER TRIGGER`不会，因为它在计划之后进行。

<h1 id="jSSy7">注意项</h1>
<h2 id="nGyaK">使用Explain analyse得到的执行时间与不使用执行时间存在偏差</h2>
有两种情况会导致这种情况的发生：

1. Explain analyse不会将数据发往客户端，因此避免了网络中的时间开销。然而实际生产中，如果数据量过大，网络时间开销会占用很多，所以两者存在差异。
2. Explain analyse本身的开销较大，比如其本身会执行一些系统调用，比如`gettimeofday()`，这些调用速度慢的话，会造成Explain analyse执行时间较长。

<h2 id="mwH6s">如果两种情况差距较大，你无法从一种情况的执行计划去推测另一种情况</h2>
比如同一张表，当规模较小时的执行计划可能与规模较大时完全不同

<h2 id="dHAGw">估计值与实际执行值有偏差</h2>
有几种情况会导致这种结果，比如使用`Limit`

以该执行计划为例：

```plain
EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000 LIMIT 2;

                                                          QUERY PLAN
-------------------------------------------------------------------​------------------------------------------------------------
 Limit  (cost=0.29..14.71 rows=2 width=244) (actual time=0.177..0.249 rows=2 loops=1)
   ->  Index Scan using tenk1_unique2 on tenk1  (cost=0.29..72.42 rows=10 width=244) (actual time=0.174..0.244 rows=2 loops=1)
         Index Cond: (unique2 > 9000)
         Filter: (unique1 < 100)
         Rows Removed by Filter: 287
 Planning time: 0.096 ms
 Execution time: 0.336 ms
```

`Index Scan using tenk1_unique2 on tenk1  (cost=0.29..72.42 rows=10 width=244) (actual time=0.174..0.244 rows=2 loops=1)`，此处估计获取的结果为10行，但由于上层`Limit`的限制，实际上只得到两行就停止了。



另一种情况是关于`Merge Join`，设想现在有两个已通过连接字段排序的结果集A和B，在`Join`过程中，A的最后一个值，也就是最大的一个值比当前比对的B结果集中的某个值更小，那么这次`Merge Join`就可以提前停止了，因为无法再找到更大的A值了。



还有一种情况同样关于`Merge Join`，如果设想A结果集为`1,1,3`；B结果集为`1,2,3`，B为内部表，在进行`Merge Join`时，`A[1]=B[1]`，因此可以连接。连接后则对比`A[2]`和`B[2]`，但是此时发现`A[2]`依然等于`1`，因此就重新扫描一遍`B`，使得`A[2]`和`B[1]`连接。这种情况会发生重复扫描。 Explain Analyse得到的结果中，可能内部表得到的实际扫描行数会比内部表行数多。

