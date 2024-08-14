---
layout: default
title:  "Golang-CSP(Communicating Sequential Processes)见解与讨论"
date:   2024-06-20 14:50:49 +0800
categories: Golang
excerpt: "Golang在现代高级编程语言中拥有一席之地，很大一部分原因是因为它在并发编程上的优势。"
---
<a name="WfAsd"></a>
# 概况
Golang在现代高级编程语言中拥有一席之地，很大一部分原因是因为它在并发编程上的优势。Golang主要并发编程思想由1978年提出的CSP(Communicating Sequential Processes)发展而来。<br />一般来说，并发编程最让人困惑的是并发安全问题，要时时刻刻注意共享变量、线程的状态，而CSP则将关注点主要放在一个过程(Process)的输入和输出，各个过程中间通过输入输出进行交互。这种注重交互的思想很大的优势在于：符合人类对现实问题的建模过程，更具有顺序性，而不是对各个变量状态的管理这种在时间尺度上不能很好体现的玩意。
<a name="AmTdF"></a>
# 运行机制
CSP包含三个核心概念：过程(Process)、通道(channel)、事件(event)。程序是一段独立的代码流程，通道是程序之间沟通的媒介，事件是通过通道交互的动作。三者配合实现了并发系统的正常工作流程，下图表示数据的归属权在多个进程间变化，并且确保了并发安全。<br />在我的理解中，这种基于CSP的并发编程，将传统并发原语(锁，信号量)在资源竞争中的时间顺序上的不确定性进行了明确，并通过规范的消息交换机制使得整个过程更容易预测和更不容易犯错。<br />![image.png](http://pic.kinghno3.top/kinghno3_blog/20240602184319.png)
<a name="bpsVI"></a>
# Golang中的CSP
CSP在Golang中主要体现在三个内置元素上，分别是Goroutine，Channel和Select。<br />goroutine对应着CSP中的过程，Channel也对应着通道，往Channel进行收发数据则对应着事件。在Golang中，我们无需关注操作系统的线程，我们要实现并发时只需要通过`go`关键字开启一个协程，调度器则会自动将这部分协程代码在合适的时机放置到合适的线程中执行，goroutine非常轻量，在使用时基本不用顾虑，除非你很少关闭它造成内存泄漏。需要顾虑的时候，说明代码架构可能需要改动了。<br />Channel则负责在Goroutine之间传输数据，其底层是并发安全的，并且Golang在CSP的基础上将Channel扩充了有缓冲区的模式，这样接收者或发送者在找不到匹配的对端goroutine时就不用阻塞了。同时，Channel也可以配合Select进行信号通知，比如超时通知、上层Goroutine下发的关闭通知等。<br />接下来用一段代码展示CSP在Golang中的应用：
```typescript
package main

import (
    "fmt"
    "time"
)

// 模拟数据处理函数
func processData(data int, done chan bool) {
    fmt.Printf("Processing data %d\n", data)
    time.Sleep(2 * time.Second) // 模拟耗时操作
    done <- true // 发送处理完成的信号
}

func main() {
    dataChannel := make(chan int) // 创建数据通道
    done := make(chan bool)       // 创建完成通道，用于通知数据处理完成

    // 启动Goroutine来处理数据
    go func() {
        // 每次循环从dataChannel中取出一个数据进行处理
        for data := range dataChannel {
            processData(data, done)
        }
    }()

    // 使用Select进行通信和超时控制
    go func() {
        for {
            // 接收三种信号
            // case1: 当dataChannel可写时，向其发送数据10
            // case2: 从done可读出数据
            // case3: 当超过超时时间时
            // 如果同时满足了多种信号，那么随机选择一个满足的信号进行处理
            select {
            case dataChannel <- 10: // 尝试向channel发送数据
                fmt.Println("Sent data to be processed")
            case <-done: // 处理完成的通知
                fmt.Println("Data processing completed")
            case <-time.After(1 * time.Second): // 超时控制
                fmt.Println("Timeout, no data received")
                return
            }
        }
    }()

    // 让主Goroutine等待，模拟运行
    time.Sleep(5 * time.Second)
}
```
在第二个goroutine中，第一次循环会触发case1，会向dataChannel发送数据10，然后进入下一次循环等待信号。<br />由于processData处理数据需要2s，还没处理完时就触发了case3，因此会打印："Timeout, no data received"，然后该goroutine退出。

如果不使用channel而要实现相同功能的条件下，我能想到的一个实现方法：需要一个变量status维护状态，用于表示当前是否可以处理数据；需要一个计时器，判断是否超时，如果处理数据，则将计时器复原重新计时，如果超时则退出函数；需要一个变量或队列存储待处理的数据。当然，都需要使用锁进行保护。另外，第二个goroutine可以使用sync.Cond通知第一个goroutine处理数据。<br />整个实现下来，关注点比使用channel时更多，比如数据并发安全、数据处理状态等。因此，像这种在多个goroutine之间编排的场景，尽量先考虑使用channel。 一般来说，通知机制使用channel的close函数即可，但是需要多次通知时，可以考虑使用sync.Cond的Broadcast。
<a name="R83Bl"></a>
# 总结
CSP将并发编程的关注点放在线程之间的交互上，也就是输入和输出。这种通信模型是在单纯的锁等传统并发原语的基础上的一层抽象，可以使开发者更容易、更自然地对现实问题进行建模，减少在死锁等问题上的精力耗费。<br />然而，对于高性能要求或线程间执行顺序要求不高的场景，用锁等原语可能会更简单。<br />最后，囿于作者水平，本文有任何理解不到位或者不完善的地方欢迎指出。
