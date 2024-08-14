---
layout: default
title:  "Goland速通_数据结构_Map"
date:   2024-05-20 14:50:49 +0800
categories: Golang
---
笔者使用的Golang版本为1.18.10

Map是面试中最常问到的Golang基础知识之一，在平时开发中也具有相当重要的地位，然而，由于其底层实现原理，在使用过程中也常常遇到各种问题，因此需要从应用到原理把Map研究透，才能更加得心应手
<a name="tuTWd"></a>
# 基础用法
<a name="IjX3T"></a>
## 初始化
Map在使用前需要初始化，否则会出现Panic，如下创建了一个Key和Value类型都为string的map对象，但是没有初始化：
```go
var m map[string]string
m["name"] = "kinghno3" // out: panic: assignment to entry in nil map
```
多种初始化方法：
```go
m1 := make(map[string]string, 10) // 使用make初始化，并设置Map容量为10
m2 := map[string]string{} // 使用大括号初始化
m3 := map[string]string{"name":"kinghno3","age":"18"} // 赋初始值
```
<a name="b2f8s"></a>
## 读写
写入Map:
```go
m["key"] = "value" // m类型为map[string]string
```
读取Map:
```go
value1 := m1["key"] // 一个接收参数，取m1中键为"key"的值，如果没有该值，则value为map值类型的零值
value2,ok := m2["key"] // 两个接收参数，value2为取到的值，ok为bool类型，true代表取到该值，false代表m2中无该值
```
<a name="EXMSL"></a>
## 拷贝
<a name="Bf1Sp"></a>
### 浅拷贝
浅拷贝则是拷贝对同一个底层Map数据结构体的引用，比如直接赋值:
```go
m1 := map[string]interface{}{"num": 1}
m2 := m1
m2["num"] = 2 // 改变m2的值同时也相当于改变了m1的值
log.Println(m1["num"]) // out: 2
```
<a name="WgWTb"></a>
### 深拷贝
深拷贝则是将底层数据也拷贝一份：
```go
m1 := map[string]interface{}{"one": 1, "two": 2, "three": 3}
m2 := make(map[string]interface{})
for k, v := range m1 {
    m2[k] = v
}
log.Println(m2) // out: map[one:1 three:3 two:2]
m2["one"] = "one" // 即便改变了m2的键值对，也不会对m1有影响
log.Println(m1) // out: map[one:1 three:3 two:2] 
log.Println(m2) // out: map[one:one three:3 two:2]
```
如果Map中的值本身就是指针，那么这种方式也只能拷贝指针，而不是拷贝最终指向的底层数据。
<a name="Zdrtm"></a>
## 遍历
Map可以通过for range对每个键值对都进行遍历:
```go
m1 := map[string]interface{}{"one": 1, "two": 2, "three": 3}
for k, v := range m1 {
    log.Printf("key: %s, value: %+v", k, v)
}
// out:
// key: three, value: 3
// key: one, value: 1
// key: two, value: 2
```
**需要注意map的遍历并不是固定顺序**。
<a name="pDrYn"></a>
## 删除
```typescript
m := map[string]string{"name": "kinghno3","age": "18"}
delete(m, "name")
log.Printf("m: %+v",m) // out: m: map[age:18]
```
<a name="lG5kE"></a>
# 数据结构
Map实际上为一个hmap，hmap的结构如下：
```go
// A header for a Go map.
type hmap struct {
	count     int // Map中元素数量
	flags     uint8
	B         uint8  // 代表桶的数量，数量为2^B
	noverflow uint16 // 溢出桶的数量，当桶较多时为一个估算值
	hash0     uint32 // 用于计算哈希值

	buckets    unsafe.Pointer // 指向桶空间的指针
	oldbuckets unsafe.Pointer // 指向旧桶的指针，当为nil时代表扩容结束
	nevacuate  uintptr        // 扩容进度，下一个要扩容的桶

	extra *mapextra // 存储溢出桶的信息
}
```
存储桶的结构如下：
```go
// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```
`tophash`为一个uint8数组，存储该桶每个Key的哈希值的高八位，用于快速匹配。<br />后面的存储空间则跟着`bucketCnt`个key和`bucketCnt`个value。key和value各自放在一起，而不是交错放置，这样就避免了key和value占用存储空间不同导致的内存对齐。内存对齐会导致空间的浪费。<br />最后的位置则放了一个bmap指针，指向一个溢出桶。溢出桶的结构和存储桶相同。<br />示意图：<br />![image.png](http://pic.kinghno3.top/kinghno3_blog/20240602183758.png)
<a name="VBchz"></a>
## 溢出桶
溢出桶的存在是为了减少Map扩容的次数，因为每次扩容都意味着要迁移大量的数据。<br />当存储桶中元素存满，但是新存入的元素依然需要存到该存储桶时，就会存入该存储桶指向的溢出桶中。
<a name="p3sHm"></a>
# 扩容
Map用于存储键值对的结构为桶，当当前桶存放量达到阈值后，将会开辟新的空间作为桶空间存储键值对。Map的扩容为**渐进式扩容**，也就是扩容动作分多次完成，当旧桶不再使用时，代表迁移完成。
<a name="wQDEa"></a>
## 扩容时机
**第一种情况**：当存储的键值对数量`count`与桶数`2^B`之比大于`loadFactor`时，就会发生扩容。loadFactor目前为`6.5`，此时的扩容策略为**翻倍扩容**。<br />相关代码：
```go
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```
**第二种情况**：当溢出桶数量>=存储桶数量时，进行扩容。此时扩容策略为**等量扩容。**这种情况会出现在每个桶的元素非常分散的时候，比如插入大量元素再删除大量元素，会导致大量的空间浪费，而且查询元素时耗时增加，因此通过等量扩容将分散的元素重新密排，提高空间利用率和查询效率。
```go
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// 调整B值，如果大于15，则桶数大于2^15，超出了uint8的范围
	if B > 15 {
		B = 15
	}
	// uint16(1)<<(B&15)为桶数
	return noverflow >= uint16(1)<<(B&15)
}
```
<a name="BC6rA"></a>
## 翻倍扩容
翻倍扩容时，当前的Bucket会挂载到hmap.oldBucket上，而新开辟的Bucket会挂载到hmap.bucket上。开辟和挂载并没有执行将数据从oldBucket迁移到bucket的过程，真正的迁移过程发生在写入、删除Map元素的时候。<br />写入和删除的元素(记作元素E)如果落在迁移过程的桶，那么会先进行迁移过程，迁移完该桶后再执行一次迁移桶操作：
```typescript
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// bucket为通过元素E.key对应的hash值算出的桶编号idx
  // bucket & h.oldbucketmask() 意为找到idx所对应的旧桶编号idxOld
  // 完成该旧桶的迁移工作
	evacuate(t, h, bucket&h.oldbucketmask())

	// 再主动迁移一个桶
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
```
等待nevacuate达到新桶的数量时，代表所有桶迁移完毕，此时将hmap.bucket置为nil，清除掉老的溢出桶。
<a name="czl6O"></a>
### 迁移规则
桶数恒为2的倍数，桶编号为`0 ~ 2^B-1`，`2^B-1`在二进制下为`111...1111`的形式。<br />假如现在桶数为8，那么一个key的hash值末尾三位为`110`的元素，应该落在`110 & 111 = 110`也就是编号为6的桶中。

**翻倍扩容时，桶6需要分流到新创建的两个桶中，如何确定这两个桶的编号呢？**<br />桶数翻倍后，最大的桶编号为15，也就是`1111`。那么在决定桶编号时就需要看末四位。<br />桶6的二进制为`110`，那么增加一位到`110`之前，桶中不同的key对应不同的hash值，因此倒数第四位可以为0也可以为1，最终末四位得到的数就为要分流到的目标桶(`0110`和`1110`)，也就是桶6和桶14.
<a name="C7Snw"></a>
## 等量扩容
等量扩容发生在溢出桶太多的时候。<br />旧桶迁移到新桶时，桶编号不会变动，但是元素迁移后会更加紧凑。![image.png](http://pic.kinghno3.top/kinghno3_blog/20240602183820.png)
<a name="II0tA"></a>
# 并发
Go原生Map不支持并发读写，需要搭配锁等并发安全机制使用。<br />或者可以使用sync.Map代替。
<a name="y4n6C"></a>
# 后续
接下来会更新Channel、Gouroutine这些在Golang中最核心的概念和结构体。<br />对Map部分有疑问、补充、修正都欢迎提出，共同学习
