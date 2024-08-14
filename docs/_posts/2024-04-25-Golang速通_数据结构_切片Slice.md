---
layout: default
title:  "Golang速通_数据结构_切片Slice"
date:   2024-04-25 14:50:49 +0800
categories: Golang
---
以更新博客与夯实基础为契机，将Golang涉及的知识过一次，内容主要包含基本用法、部分底层原理、易错点。<br />**目标**： 尽量简洁、清晰，掌握后单知识点上能够应对绝大部分面试。<br />笔者使用的Golang版本位1.18.10

Slice是Go中最常用的数据结构之一，用于存储同一类元素集合，而且其长度可以动态扩充。
<a name="bAVUr"></a>
# 数据结构
切片是对数组的扩展，一个切片对象实际上是一个slice结构体，slice结构体为：
```go
type slice struct{
    array unsafe.Pointer // 指向存储数据的数组空间
    len int // 切片元素个数
    cap int // 切片容量
}
```
<a name="TTaDa"></a>
# 用法
<a name="kaKv5"></a>
## 初始化
可以通过原语获得一个Slice对象的长度或容量，长度是Slice中元素的个数，容量则是Slice能够容纳的元素个数。
```go
s := make([]int,5,8) // 创建一个int切片，初始化了5个为0的元素，并且切片容量为8
s = append(s,1) // 第六个元素为1
l := len(s) // 获取长度，l为6
c := cap(s) // 获取容量，c为8
```
<a name="P6VVq"></a>
## 截取
在开发过程中，截取是非常方便的功能，写法简洁，功能强大，可以取到一个切片的合法片段。
```go
s := []int{1,2,3,4,5,6,7,8} // 创建一个元素为1~8的切片

s1 := s[2:5] // 从下标为2的元素开始取，取到第(5-1)个元素，s1为[]int{3,4,5}

s2 := s[8:8] // s2为一个[]int类型的空切片

// 9是一个非法位置，
// 运行出现Panic:slice bounds out of range [:9] with capacity 8
s3 := s[9:9] 
```
截取下来的切片实际上和原切片使用的是同一个数组，可以理解为对同一个数组的不同视口：
```go
s := []int{1, 2, 3, 4, 5, 6, 7, 8} // 创建一个元素为1~8的切片
s1 := s[0:3] // {1,2,3}
s2 := s[2:5] // {3,4,5}

// s: 0xc0000c4080, s1: 0xc0000c4080, s2: 0xc0000c4090
log.Printf("s: %p, s1: %p, s2: %p", s, s1, s2)

s1[1] = 666
s2[0] = 888
log.Println(s) // {1,666,888,4,5,6,7,8}
```
示意图：<br />![image.png](http://pic.kinghno3.top/kinghno3_blog/20240602183555.png)

但是如果在取得s1后再对s进行扩容，导致数组的地址修改，那么s和s1就不会共用一个底层数组：
```go
s := []int{1, 2, 3, 4, 5, 6, 7, 8} // 创建一个元素为1~8的切片
s1 := s[0:3]

s = append(s, 1) // 扩容导致底层数组拷贝到其他位置

// out: s: 0xc0000d4080, s1: 0xc0000c4080
log.Printf("s: %p, s1: %p", s, s1)

// s1内元素改变不会影响s的元素
s1[0] = 100000

log.Println(s, s1) // out: [1 2 3 4 5 6 7 8 1] [100000 2 3]
```

<a name="tkxTl"></a>
# 扩容
扩容的几种方式：
```go
s := []int{}
s = append(s,1) // 扩容一个元素
s = append(s,1,2,3) // 扩容多个元素
s = append(s,[]int{4,5,6}...) // 解构切片{4,5,6}为4，5，6三个参数，并传入append
```

当append添加的元素放不下切片时，就会触发扩容，扩容规则比较复杂，每次扩容时都会扩一些多余的空间，作为余量，避免频繁扩容。但是这个余量是有规则的。

- **当原容量的2倍可以满足append需求时**

在小于256个元素时，扩容是将容量翻倍。<br />当大于256个元素时，将旧容量扩大1.25倍并加上256*(3/4)，如果依然不足以容纳新元素，则继续扩大1.25倍并加上256*(3/4)。由于内存对齐的原因，会在新容量基础上再增大。<br />贴上go1.18.10的部分扩容代码：
```go
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		const threshold = 256
		if old.cap < threshold {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				// Transition from growing 2x for small slices
				// to growing 1.25x for large slices. This formula
				// gives a smooth-ish transition between the two.
				newcap += (newcap + 3*threshold) / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
```

- **不满足时**

首先容量需要达到所需容量，然后再根据内存分配机制进行增大。<br />代码案例如下：
```go
s := []int{1,2} // len: 2, cap: 2
s = append(s,4,5,6) // len: 5, cap: 6
```
读者可能对cap为6不解，这里直接说扩容过程：<br />append传入了三个元素，因此cap至少需要5，按理来说40Byte就足够<br />但是根据golang内存分配机制，能够获取到的内存为48Byte。<br />一个Int占8个Byte，因此cap = `48/8 = 6`<br />具体内存分配机制以后进行解析

<a name="g1SiJ"></a>
# 作为函数参数
由于切片对象实际上是一个Slice结构体，其中包含指向底层数组的指针。由于Go仅支持值传递，函数中使用的是切片的副本，但这些副本指向同一个底层数组。因此，在函数内部修改切片的元素会影响原切片。<br />但是，依然要注意，如果因为函数中对切片扩容，造成底层数组地址变化，则函数内外的切片对象指向不同区域。
```go
func main() {
	s := []int{1, 2, 3, 4, 5}
	testFunc(s)
	log.Println(s) // out: [1 2 10000 4 5]
}

func testFunc(s []int) {
	s[2] = 10000 // out: [1 2 10000 4 5]
	s = append(s, 6) // 扩容后底层数组地址变化
	s[2] = 999 // out: [1 2 999 4 5 6]
}
```
如果要使扩容后，函数内外的切片都指向同一个底层数组，那么可以直接给参数传递切片指针：
```go
func testFunc(s *[]int){
    *s = append(*s,6)
    *s = append(*s,7)
}
```
<a name="Sddfb"></a>
# 后续
接下来会更新Map、Channel、Gouroutine这些在Golang中最核心的概念和结构体。<br />对Slice部分有疑问、补充、修正都欢迎提出，共同学习



