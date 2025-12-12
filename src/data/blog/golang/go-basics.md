---
author: Flobby
pubDatetime: 2024-07-30T10:00:00Z
title: Go 语言新手村：变量、常量和枚举用法
slug: go-basics
featured: false
draft: false
tags:
  - Golang
  - 入门教程
description: Go语言基础教程，介绍变量、常量、枚举以及作用域等核心概念。
---

## 1. 介绍

### 1.1 发展

2007年，Google开始着手开发Go语言。

2009年11月，Google发布了Go语言的第一个稳定版本，即Go 1.0。这标志着Go语言正式面世，并开始逐渐受到广泛关注和应用。

2018年8月，Go 1.11发布，引入了模块支持、WebAssembly支持、延迟函数调用等特性，这些特性使得Go语言在项目管理、跨平台应用和异步编程等方面有了更加强大的能力。

2021年2月，Go 1.16发布，加入了对模块和错误处理的改进、垃圾回收器的性能提升等特性。

### 1.2 特点

+ **高效性**：Go语言具有高效的编译速度、并发处理能力和内存管理机制，适合处理高并发、高吞吐量的场景。
+ **并发性**：Go语言提供了原生的并发编程模型，通过协程（goroutine）和通道（channel）实现了轻量级的线程管理和通信。
+ **安全性**：Go语言内置了安全性特性，例如强类型、空指针检查、内存自动管理等。
+ **跨平台**：Go语言支持多种操作系统和CPU架构，可以轻松地在不同平台之间进行开发和部署。

## 2. 基础概念

### 2.1 内建变量类型

Go 语言中内建变量类型分为三种，分别是布尔类型、数值类型和字符串类型

1. **布尔类型**：布尔类型的关键字是`bool`，取值范围是`true 或 false`，其中默认值为`false`
2. **数值类型**：Go 语言中数值类型很多，包括整数、浮点数、复数。

|      类型      |          描述          |    大小    |
| :------------: | :--------------------: | :--------: |
|     `int`      |   平台相关有符号整数   | 32位或64位 |
|     `int8`     |     8位有符号整数      |    8位     |
|    `int16`     |     16位有符号整数     |    16位    |
|    `int32`     |     32位有符号整数     |    32位    |
|    `int64`     |     64位有符号整数     |    64位    |
|     `uint`     |   平台相关无符号整数   | 32位或64位 |
| `uint8` (byte) |     8位无符号整数      |    8位     |
|    float32     |      单精度浮点数      |    32位    |
|    float64     |      双精度浮点数      |    64位    |
|   complex64    | 实部虚部分别32位的复数 |    64位    |
|   complex128   | 实部虚部分别64位的复数 |   128位    |

3. **字符串类型**：关键字`string`，默认值为空字符串

在 Go 语言中还有两个特殊的类型，`rune` 和 `byte`。rune 是 int32 的别名，常用于处理文本字符；byte 是 uint8 的别名，常用于处理字节、二进制文本。

### 2.2 变量

Go 语言中使用 var 关键字定义变量：

```go
func main() {
    var a int
    var b string
    var c bool
    // 输出结果是该类型的零值（默认值）
    fmt.Printf("a = %d, b = %q, c = %t \n", a, b, c)

    a = 100
    b = "hello"
    c = true
    fmt.Printf("a = %d, b = %q, c = %t \n", a, b, c)

    var name string = "张三"
    var age int = 18
    fmt.Printf("name = %s, age = %d \n", name, age)

    // 类型推导
    var country = "中国"
    var price = 100
    fmt.Printf("country = %s, price = %d \n", country, price)

    // 短变量声明
    year := 2025
    province, city := "江苏省", "苏州市"
    fmt.Printf("year = %d, province = %s, city = %s \n", year, province, city)

    // 匿名变量
    x, _ := anonymous()
    fmt.Println(x)
}

func anonymous() (string, int) {
    return "李四", 20
}
```

**核心概念：**

1. **零值初始化**：Go为所有基础类型提供零值保障，避免未初始化风险
2. **类型推导**：声明时省略变量类型且立刻赋值，编译器根据赋值自动推断类型
3. **短变量声明**：在函数内部快速声明并初始化变量的方式 `year := 2025`
4. **匿名变量**：使用 `_` 进行占位，匿名变量不占用命名空间，不会分配内存

### 2.3 常量

常量使用 `const` 关键字进行声明。常量是**在编译时就确定其值**，并且在程序运行期间**不可更改**的标识符。

#### 枚举

在 Go 语言中，可以通过使用 `const` 关键字结合 `iota` 标识符实现枚举功能。

`iota` 是 Go 语言中的一个特殊标识符，仅在 `const` 声明块中有效。它代表了从 0 开始的连续整数值。

```go
const (
  Sunday    = iota // 0
  Monday           // 1
  Tuesday          // 2
  Wednesday        // 3
  Thursday         // 4
  Friday           // 5
  Saturday         // 6
)
```

**指定起始值**：

```go
const (
    a = iota + 1 // 1
    b            // 2
    c            // 3
)
```

**跳过某些值**：

```go
const (
    _ = iota // 忽略第0个值
    a        // 1
    _        // 忽略第2个值
    b        // 3
)
```

**位操作**

```go
const (
    b = 1 << (10 * iota)  // 1
    kb                     // 1024
    mb                     // 1048576
    gb                     // 1073741824
)
```

### 2.4 作用域

Go 语言存在全局和局部两种作用域，其中全局作用域在Go语言中也被称为`包级作用域`。

**包级作用域 (Package-Level Scope)**

- **位置**：在任何函数、方法或代码块之外的顶层声明
- **生命周期**：从程序启动时创建，到程序结束时销毁

**局部作用域 (Local Scope)**

- **位置**：在函数内部、方法内部或代码块内部声明
- **生命周期**：从声明/初始化点开始，到包含它的函数、方法或代码块执行结束时终止

#### 包级变量的导出

在 Go 语言中，如果变量的名字以**大写字母开头**，则可以被其他包通过导入的方式访问。

```go
package other

// 可导出的包级变量
var ExportedVar = "This is an exported variable"

// 不可导出的变量
var unexportedVar = "This is a unexported variable"
```

```go
package main

import (
    "fmt"
    "go-learning/other"
)

func main() {
    fmt.Printf("可导出变量：%s \n", other.ExportedVar)
    // other.unexportedVar 无法访问
}
```
