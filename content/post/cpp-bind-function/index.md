---
title: "C++ std::bind 与 std::function 用法详解"
description: "深入理解 C++ 中 std::bind 和 std::function 的使用方法，包括绑定成员函数、自由函数、参数占位符以及实际应用场景。"
date: 2020-03-18
categories:
    - 技术文章
tags:
    - C++
    - STL
    - 函数式编程
draft: false
---

## 概述

`std::function` 和 `std::bind` 是 C++ 中实现**函数对象包装**和**参数绑定**的核心工具。它们最初在 `std::tr1` 命名空间中引入（TR1 技术报告），C++11 后正式纳入标准库。

| 组件 | 作用 | 头文件 |
|------|------|--------|
| `std::function` | 通用函数包装器，可存储任意可调用对象 | `<functional>` |
| `std::bind` | 将函数与部分参数绑定，生成新的可调用对象 | `<functional>` |

> **注意**：`std::tr1::bind` / `std::tr1::function` 是旧写法，C++11 及以后应直接使用 `std::bind` / `std::function`。

---

## 1. std::function 基础

`std::function` 是一个通用的函数包装器，可以存储普通函数、Lambda、函数对象以及成员函数绑定后的结果。

### 1.1 声明语法

```cpp
#include <functional>

// 模板参数为函数签名：返回值类型(参数类型列表)
std::function<void()> f1;              // 无参无返回值
std::function<int(int, int)> f2;       // 两个 int 参数，返回 int
std::function<std::string(const std::string&)> f3;  // 字符串参数和返回值
```

### 1.2 存储不同类型的可调用对象

```cpp
// 存储普通函数
void greet() { std::cout << "Hello!" << std::endl; }
std::function<void()> f1 = greet;

// 存储 Lambda
std::function<int(int, int)> f2 = [](int a, int b) { return a + b; };

// 存储函数对象（仿函数）
struct Multiply {
    int operator()(int a, int b) { return a * b; }
};
std::function<int(int, int)> f3 = Multiply();
```

---

## 2. std::bind 绑定函数

`std::bind` 的核心用途是将函数与部分（或全部）参数绑定，生成一个新的可调用对象。这在回调、事件处理等场景中非常有用。

### 2.1 绑定语法

```cpp
auto 新函数 = std::bind(&要调用的函数, 参数1, 参数2, ..., _1, _2, ...);
```

- 固定参数：直接传值
- 延迟参数：使用占位符 `std::placeholders::_1`、`_2`、`_3` 等

### 2.2 绑定成员函数

成员函数不能直接作为普通函数指针传递，需要通过 `std::bind` 绑定对象实例：

```cpp
class A {
public:
    void test() {
        std::cout << "A::test called" << std::endl;
    }

    void print(int value) {
        std::cout << "value = " << value << std::endl;
    }
};

A a;

// 绑定无参成员函数：传入函数指针 + 对象地址
std::function<void()> f1 = std::bind(&A::test, &a);
f1();  // 输出: A::test called

// 绑定带参成员函数：_1 表示调用时传入的第一个参数
std::function<void(int)> f2 = std::bind(&A::print, &a, std::placeholders::_1);
f2(42);  // 输出: value = 42
```

### 2.3 绑定自由函数

```cpp
int add(int a, int b) {
    return a + b;
}

// 绑定第一个参数为 10，第二个参数延迟传入
auto add10 = std::bind(add, 10, std::placeholders::_1);
std::cout << add10(5) << std::endl;  // 输出: 15

// 交换参数顺序
auto swap_add = std::bind(add, std::placeholders::_2, std::placeholders::_1);
std::cout << swap_add(1, 2) << std::endl;  // 等价于 add(2, 1)，输出: 3
```

---

## 3. 占位符详解

占位符 `std::placeholders::_N` 表示绑定后的函数被调用时的第 N 个参数：

```cpp
void foo(int a, int b, int c) {
    std::cout << a << " " << b << " " << c << std::endl;
}

// _1 → 调用时第1个参数映射到 foo 的第3个参数
// _2 → 调用时第2个参数映射到 foo 的第1个参数
// 100 → foo 的第2个参数固定为 100
auto bound = std::bind(foo, std::placeholders::_2, 100, std::placeholders::_1);
bound(1, 2);  // 等价于 foo(2, 100, 1)，输出: 2 100 1
```

---

## 4. 实际应用场景

### 4.1 回调函数

```cpp
class Button {
public:
    std::function<void()> onClick;

    void click() {
        if (onClick) onClick();
    }
};

class Controller {
public:
    void handleClick() {
        std::cout << "Button clicked!" << std::endl;
    }
};

Controller ctrl;
Button btn;
btn.onClick = std::bind(&Controller::handleClick, &ctrl);
btn.click();  // 输出: Button clicked!
```

### 4.2 与 STL 算法配合

```cpp
#include <algorithm>
#include <vector>

std::vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8};

// 找出所有大于 4 的元素
auto it = std::find_if(nums.begin(), nums.end(),
    std::bind(std::greater<int>(), std::placeholders::_1, 4));
```

---

## 5. C++11 之后的替代方案

在现代 C++ 中，**Lambda 表达式**在大多数场景下可以替代 `std::bind`，且可读性更好：

```cpp
// std::bind 写法
auto f = std::bind(&A::print, &a, std::placeholders::_1);

// Lambda 写法（推荐）
auto f = [&a](int value) { a.print(value); };
```

| 对比 | `std::bind` | Lambda |
|------|-------------|--------|
| 可读性 | 参数多时较难理解 | 直观清晰 |
| 灵活性 | 固定模式 | 可包含任意逻辑 |
| 性能 | 可能有额外开销 | 通常可内联优化 |
| 适用场景 | 简单参数绑定 | 通用场景 |

> **建议**：新代码优先使用 Lambda，仅在需要兼容旧代码或特定参数重排时使用 `std::bind`。
