---
title: "Go 并发模式：从 Goroutine 到 Channel 的实践指南"
description: "深入理解 Go 语言并发原语，通过实际案例掌握常用并发模式"
date: 2024-03-15
lastmod: 2024-03-15
image: "cover.jpg"
categories:
    - 技术文章
tags:
    - Go
    - 并发
    - 后端开发
draft: false
---

Go 语言的并发模型是其最核心的特性之一。本文通过实际案例，介绍几种常用的 Go 并发模式。

## 基础：Goroutine 与 Channel

Goroutine 是 Go 的轻量级线程，Channel 是它们之间通信的管道。

```go
func main() {
    ch := make(chan int, 1)
    
    go func() {
        ch <- heavyComputation()
    }()
    
    result := <-ch
    fmt.Println(result)
}
```

## 模式一：Fan-Out / Fan-In

将一个任务分发给多个 Worker，再收集结果：

```go
func fanOut(input <-chan int, workers int) []<-chan int {
    outputs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = worker(input)
    }
    return outputs
}

func fanIn(inputs ...<-chan int) <-chan int {
    merged := make(chan int)
    var wg sync.WaitGroup
    
    output := func(ch <-chan int) {
        defer wg.Done()
        for val := range ch {
            merged <- val
        }
    }
    
    wg.Add(len(inputs))
    for _, ch := range inputs {
        go output(ch)
    }
    
    go func() {
        wg.Wait()
        close(merged)
    }()
    
    return merged
}
```

## 模式二：Pipeline

将数据处理分成多个阶段，每个阶段由独立 Goroutine 处理：

```go
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}
```

## 模式三：Context 控制取消

使用 `context` 优雅地控制 Goroutine 生命周期：

```go
func doWork(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // 执行实际工作
            if err := processItem(); err != nil {
                return err
            }
        }
    }
}

// 调用方
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := doWork(ctx); err != nil {
    log.Printf("work failed: %v", err)
}
```

## 总结

| 模式 | 适用场景 | 注意事项 |
|------|----------|----------|
| Fan-Out/In | CPU 密集型并行任务 | 控制 Worker 数量避免资源耗尽 |
| Pipeline | 流式数据处理 | 确保每个阶段正确关闭 Channel |
| Context | 需要超时/取消控制 | 始终传递 Context，不要存储 |

Go 的并发哲学是：**不要通过共享内存来通信，而要通过通信来共享内存**。掌握这些模式，能让你写出更安全、更高效的并发代码。

## 参考资源

- [Go Blog: Concurrency Patterns](https://go.dev/blog/pipelines)
- [Effective Go: Concurrency](https://go.dev/doc/effective_go#concurrency)
- 《Go 并发之道》— Katherine Cox-Buday
