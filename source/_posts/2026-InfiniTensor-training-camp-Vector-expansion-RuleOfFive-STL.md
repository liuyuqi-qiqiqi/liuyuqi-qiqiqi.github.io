---
title: 2026年夏季InfiniTensor训练营｜Vector扩容实现、C++三大/五大法则、继承多态与现代C++特性
date: 2026-07-12 16:00:00
tags: 2026训练营,InfiniTensor,C++,Vector扩容,RuleofThree,RuleofFive,STL容器,继承多态,智能指针
categories: 2026夏季InfiniTensor训练营
cover:
description: 本节课在手写Vector基础上实现自动扩容功能，详解双重释放问题、构造函数初始化、C++三法则与五法则、移动语义、STL容器特性、继承与多态、抽象类、智能指针以及auto、using现代关键字。
---

# 🎯 2026年夏季InfiniTensor训练营｜Vector扩容实现、C++三大/五大法则、继承多态与现代C++特性

[上一篇文章](https://liuyuqi-qiqiqi.github.io/2026/07/10/2026-InfiniTensor-training-camp-Vector-OOP-templates/) 我们从零手写了一个简易 Vector 动态数组，打通了动态内存管理、面向对象编程、运算符重载、文件分离编译和 C++ 模板类这几条核心命脉。那个 Vector 已经能跑、能扩容、能通过 Valgrind 检测——**但它离「工程级」还有一段距离**。

今天这篇文章，我们就在手写 Vector 的基础上继续深挖：完善自动扩容逻辑、搞懂那些让人头疼的「双重释放」和「浅拷贝」问题、系统学习 C++ **三大法则（Rule of Three）**与**五大法则（Rule of Five）**、掌握移动语义的精髓，然后顺势进入 STL 标准容器体系、面向对象高阶特性（继承、多态、抽象类）、智能指针以及 `auto` / `using` 等现代 C++ 关键字。内容密度拉满，但每一节都有完整可运行的代码——打开 WSL，我们开干！

---

## 📈 第二部分：基于原有 Vector 类实现自动扩容功能

### 2.1 原有静态容量的缺陷回顾

[上一篇文章](https://liuyuqi-qiqiqi.github.io/2026/07/10/2026-InfiniTensor-training-camp-Vector-OOP-templates/) 的 Vector 已经实现了基本的扩容——`pushBack` 时如果 `size >= capacity` 就调用 `resize()` 翻倍容量。但那个实现还比较「粗糙」，我们来系统性地分析一下**为什么静态容量不够用**，以及扩容背后的完整逻辑。

静态数组的核心痛点我们已经讲过——**大小必须在编译时确定**。但即便我们用 `new[]` 在堆上动态分配了数组、绕开了编译期限制，如果我们只分配一次、之后不再调整容量，实际效果和静态数组没有本质区别：

```cpp
// ❌ 伪动态：虽然用了 new，但容量写死，和静态数组没有本质区别
class FakeVector {
    int *data;
    static const int MAX = 100;   // 写死 100
public:
    FakeVector() { data = new int[MAX]; }
    void pushBack(int val) {
        if (size >= MAX) {
            cout << "满了！装不下了！" << endl;
            return;               // 直接拒绝，用户：？？？
        }
        data[size++] = val;
    }
};
```

在大模型推理中，一个请求可能产生几十万个中间张量。如果你的容器说「最多装 10000 个，多了不伺候」——那这个框架根本没法用。**真正的动态数组，必须能随着数据量的增长自动调整容量**。

### 2.2 size 与 capacity 的概念区分

在深入扩容逻辑之前，先彻底搞清两个容易混淆的概念：

| 概念 | 变量名 | 含义 | 类比 |
|------|--------|------|------|
| **大小（size）** | `size` | 当前实际存放的元素数量 | 杯子里**已经倒了多少水** |
| **容量（capacity）** | `capacity` | 当前已分配的内存最多能存多少个元素 | 杯子**最多能装多少水** |

关键关系：**`0 ≤ size ≤ capacity`** 永远成立。`size == capacity` 表示「杯子满了，再倒水需要换更大的杯子」；`size < capacity` 表示「还有空余，可以直接倒」。

```cpp
// 用代码直观感受 size 和 capacity 的区别
Vector<int> v(4);          // capacity = 4, size = 0   → 空杯子，能装 4 份
v.pushBack(10);            // capacity = 4, size = 1   → 装了 1 份，还有 3 个空位
v.pushBack(20);            // capacity = 4, size = 2
v.pushBack(30);            // capacity = 4, size = 3
v.pushBack(40);            // capacity = 4, size = 4   → 满了！
v.pushBack(50);            // 触发扩容 → capacity = 8, size = 5
```

### 2.3 动态扩容的完整逻辑实现

扩容的核心流程分为**四步**，每一步都不能出错：

```
1. 分配一块更大的新内存（通常是原容量的 2 倍）
2. 把旧内存里的所有元素拷贝到新内存
3. 释放旧内存
4. 将 data 指针指向新内存，更新 capacity
```

为什么是 **2 倍扩容**而不是每次只加 1？这是**均摊时间复杂度**的经典结论：如果每次扩容翻倍，`n` 次 `pushBack` 的总拷贝次数是 `O(n)`，平均每次 `pushBack` 是 `O(1)`。如果每次只加固定大小（比如 +10），总拷贝次数会退化到 `O(n²)`——加 10000 个元素要拷贝几百万次，性能直接崩盘。

下面给出待完善的 Vector 基础框架（后续章节会逐步补充三/五法则）：

```cpp
// Vector_v1.h —— 具备基本扩容功能的基础版本
#ifndef VECTOR_V1_H
#define VECTOR_V1_H

#include <iostream>
#include <stdexcept>
using namespace std;

template <typename T>
class Vector {
private:
    T *data;          // 指向堆上动态数组的指针
    int size;         // 当前元素数量
    int capacity;     // 当前已分配容量

    // ========== 扩容核心函数 ==========
    void resize() {
        // 步骤 1：计算新容量（首次从 0 开始，特殊处理）
        int newCapacity = (capacity == 0) ? 1 : capacity * 2;

        // 步骤 2：在堆上分配更大的新数组
        T *newData = new T[newCapacity];

        // 步骤 3：将旧数组中的有效元素逐一拷贝到新数组
        for (int i = 0; i < size; i++) {
            newData[i] = data[i];
        }

        // 步骤 4：释放旧数组的内存
        delete[] data;

        // 步骤 5：更新指针和容量
        data = newData;
        capacity = newCapacity;

        cout << "⚡ Vector 扩容："
             << (capacity / 2) << " → " << capacity
             << "（当前 size = " << size << "）" << endl;
    }

public:
    // ========== 构造函数 ==========
    Vector() : data(nullptr), size(0), capacity(0) {}

    Vector(int initCap) : size(0), capacity(initCap) {
        data = new T[capacity];
    }

    // ========== 析构函数 ==========
    ~Vector() {
        delete[] data;
        data = nullptr;
    }

    // ========== 添加元素（触发扩容的入口） ==========
    void pushBack(const T &value) {
        if (size >= capacity) {
            resize();        // 满了 → 扩容
        }
        data[size] = value;
        size++;
    }

    // ========== 下标访问 ==========
    T &operator[](int index) {
        return data[index];
    }

    const T &operator[](int index) const {
        return data[index];
    }

    // ========== 带边界检查的安全访问 ==========
    T &at(int index) {
        if (index < 0 || index >= size) {
            throw out_of_range("Vector::at() 下标越界！");
        }
        return data[index];
    }

    // ========== 获取属性 ==========
    int getSize() const     { return size; }
    int getCapacity() const { return capacity; }
    bool isEmpty() const    { return size == 0; }

    // ========== 打印调试信息 ==========
    void printInfo() const {
        cout << "Vector: size=" << size
             << ", capacity=" << capacity
             << ", elementSize=" << sizeof(T)
             << " bytes" << endl;
    }

    void printAll() const {
        cout << "[";
        for (int i = 0; i < size; i++) {
            cout << data[i];
            if (i < size - 1) cout << ", ";
        }
        cout << "]" << endl;
    }
};

#endif   // VECTOR_V1_H
```

### 2.4 扩容时的内存重新分配、数据迁移与旧内存释放流程详解

上面的 `resize()` 函数看起来只有几行，但背后涉及的内存操作非常精细。我们画一张图来理解（用容量 4 → 8 的扩容为例）：

```
扩容前：
  data ──→ [10][20][30][40]    ← 旧数组，capacity=4，size=4
            ↑                  ← data 指向这里

扩容步骤：
  Step 1: newData = new T[8]   → 堆上出现一块新内存：[ ][ ][ ][ ][ ][ ][ ][ ]
  Step 2: 逐元素拷贝           → 新内存变为：[10][20][30][40][ ][ ][ ][ ]
  Step 3: delete[] data        → 旧内存被回收：[已释放]
  Step 4: data = newData       → data 指向新数组
  Step 5: capacity = 8

扩容后：
  data ──→ [10][20][30][40][ ][ ][ ][ ]    ← 新数组，capacity=8，size=4
```

> ⚠️ **步骤顺序绝对不能乱**。最常见的错误是——**先 `delete[] data` 再拷贝数据**。一旦先释放了旧内存，`data[i]` 就已经是「已回收的垃圾内存」了，拷贝出来的内容是不可预测的随机值，程序必然出错。正确的顺序永远是：**①分配新内存 → ②拷贝数据 → ③释放旧内存 → ④更新指针**。

---

## 💥 第三部分：常见内存错误——双重释放问题详解

### 3.1 双重释放的产生场景

**双重释放（Double Free）**是指对同一块堆内存执行了两次 `delete` 或 `delete[]`。这是 C++ 中最危险的内存错误之一——编译器不会报错，程序的行为完全不可预测：可能当场崩溃，可能悄无声息地损坏了堆结构然后在几百行之后才炸，也可能这次运行正常、下次就崩。

最简单的双重释放场景：

```cpp
int *p = new int(42);
delete p;      // ✅ 第一次释放，正常
delete p;      // ❌ 第二次释放同一块内存 → 双重释放！程序大概率崩溃
```

但在实际工程中，双重释放往往不是这么明晃晃地写两次 `delete`——它常常隐藏在**拷贝**行为里。

### 3.2 浅拷贝导致的内存隐患

这是 C++ 初学者最容易踩的坑，也是双重释放最隐蔽的来源。来看一个场景：

```cpp
#include <iostream>
using namespace std;

class ShallowArray {
public:
    int *data;
    int size;

    ShallowArray(int s) : size(s) {
        data = new int[size];
    }

    ~ShallowArray() {
        delete[] data;
        cout << "析构：释放了 data 指向的内存" << endl;
    }

    // ⚠️ 没有自定义拷贝构造函数和赋值运算符！
    // 编译器会生成默认版本——逐成员复制（浅拷贝）
};

int main() {
    ShallowArray a(5);
    a.data[0] = 100;

    ShallowArray b = a;    // 调用默认拷贝构造 → 浅拷贝！
    // 此时：a.data 和 b.data 指向堆上**同一块内存**

    cout << "a.data[0] = " << a.data[0] << endl;   // 100
    cout << "b.data[0] = " << b.data[0] << endl;   // 100

    b.data[0] = 999;  // 通过 b 修改
    cout << "a.data[0] = " << a.data[0] << endl;   // 999（a 也被影响了！）

    // main 结束 → a 析构，delete[] data → 内存被释放
    //           → b 析构，delete[] data → ❌ 对同一块内存再次释放！
    //           → 程序崩溃 / 未定义行为

    return 0;
}
```

用图来理解浅拷贝的致命问题：

```
浅拷贝（默认拷贝构造）：
  a.data ──→ [100][ ][ ][ ][ ]    ← 堆上的一块内存
  b.data ──→ [100][ ][ ][ ][ ]    ← 指向同一块！

  a 析构时：delete[] a.data → 内存被回收
  b 析构时：delete[] b.data → ❌ 这块内存已经被回收了，再次释放 → 双重释放！

深拷贝（正确做法）：
  a.data ──→ [100][ ][ ][ ][ ]    ← 堆上的内存 A
  b.data ──→ [100][ ][ ][ ][ ]    ← 堆上的内存 B（独立分配、内容相同）

  a 析构时：delete[] a.data → 释放内存 A ✅
  b 析构时：delete[] b.data → 释放内存 B ✅   各管各的，互不影响
```

### 3.3 报错原因与工程规避方案

当双重释放发生时，你可能会看到类似这样的错误信息：

```
free(): double free detected in tcache 2
Aborted (core dumped)

# 或 Windows 下：
# HEAP CORRUPTION DETECTED
```

这些是操作系统堆管理器的保护机制在起作用——它在释放内存时会检查这块内存的「元数据」，如果发现它已经被标记为「已释放」，就会立刻终止程序，防止更严重的内存损坏。

**工程上的规避方案（这也正是接下来要讲的三大法则）**：

| 方案 | 说明 | 适用场景 |
|------|------|---------|
| **自定义拷贝构造函数** | 创建新对象时，分配独立内存并逐元素拷贝 | `Vector b = a;` 或 `Vector b(a);` |
| **自定义赋值运算符** | 赋值时，先释放旧资源，再分配新内存并拷贝 | `b = a;`（两个已存在的对象） |
| **禁止拷贝** | 将拷贝构造和赋值运算符声明为 `= delete` | 单例模式、独占资源的类 |
| **使用智能指针** | 用 `unique_ptr` / `shared_ptr` 管理堆内存，自动处理拷贝和释放 | 现代 C++ 推荐方式（后面会讲） |

> 🎯 **核心守则**：只要你的类里有指向堆内存的指针，就**必须**考虑「拷贝的时候指针怎么办」这个问题。无视它=埋雷，迟早会在生产环境炸。

---

## 🏗️ 第四部分：构造函数初始化列表

### 4.1 初始化列表语法规范

[上一篇文章](https://liuyuqi-qiqiqi.github.io/2026/07/10/2026-InfiniTensor-training-camp-Vector-OOP-templates/) 3.3 节我们提过一嘴初始化列表，但没有深入展开。这一节我们来系统性地把它搞透。

**初始化列表**是构造函数参数列表后面、函数体前面的一段特殊代码，用冒号 `:` 开头、逗号分隔：

```cpp
class Student {
private:
    string name;
    int id;
    double score;

public:
    // ✅ 初始化列表写法：冒号后直接初始化每个成员
    Student(string n, int i, double s)
        : name(n), id(i), score(s) {
        // 函数体里可以什么都不写，也可以写额外逻辑
        cout << "学生 " << name << " 已注册！" << endl;
    }

    // ❌ 函数体内赋值写法：先默认初始化，再覆盖
    // Student(string n, int i, double s) {
    //     name = n;    // 这里 name 早已被默认构造成了空串，再赋值 = 多余的工作
    //     id = i;
    //     score = s;
    // }
};
```

### 4.2 初始化与赋值的本质区别

这两者不是「写法不同但效果一样」的关系——它们在底层有本质区别：

```cpp
// ===== 初始化 vs 赋值的底层差异 =====

// 初始化（Initialization）：对象诞生时直接赋予初值
string name = "张三";        // 一步到位：在内存中构造 "张三"

// 赋值（Assignment）：对象已经存在，覆盖其内容
string name;                 // 第一步：默认构造，name 是空串 ""
name = "张三";               // 第二步：把空串覆盖为 "张三"——多了一步无用功
```

对于 `int`、`double` 这类基础类型，初始化和赋值的成本几乎一样——都是往内存里写一个值。但对于 `string`、`vector`、自定义类等复杂类型，**初始化比赋值的开销小**——因为初始化一步搞定，赋值则需要先默认构造、再覆盖，多做了一次无用功。

| 对比维度 | **初始化列表（Initialization）** | **函数体赋值（Assignment）** |
|---------|-------------------------------|---------------------------|
| 执行步骤 | 1 步：直接构造 | 2 步：①默认构造 ②赋值覆盖 |
| 性能 | 更高效（对复杂类型尤其明显） | 多了一次默认构造的开销 |
| `const` 成员 | ✅ 必须用初始化列表 | ❌ 编译报错，`const` 不能被赋值 |
| 引用成员 | ✅ 必须用初始化列表 | ❌ 编译报错，引用必须在定义时绑定 |
| 没有默认构造的成员 | ✅ 只能靠初始化列表 | ❌ 函数体赋值前会先尝试默认构造 → 编译失败 |

### 4.3 复杂成员、const 成员、引用成员的正确初始化方式

某些类型的成员变量**只能用初始化列表**来初始化，用函数体赋值会直接编译报错：

```cpp
#include <iostream>
#include <string>
using namespace std;

class Config {
private:
    const string APP_NAME;       // const 成员：一旦初始化就不能再改
    const int MAX_RETRY;         // const 基础类型
    int &logLevel;               // 引用成员：必须绑定到一个外部变量

public:
    // ✅ 正确的初始化方式：全部在初始化列表里完成
    Config(string name, int retry, int &level)
        : APP_NAME(name), MAX_RETRY(retry), logLevel(level) {
        // 函数体里不能写：
        // APP_NAME = name;     // ❌ const 不能被赋值！
        // logLevel = level;    // ❌ 引用不能被重新绑定！
        cout << "配置初始化完成：" << APP_NAME << endl;
    }

    // ❌ 错误写法——编译直接报错
    // Config(string name, int retry, int &level) {
    //     APP_NAME = name;    // ❌ 编译错误：不能给 const 成员赋值
    //     MAX_RETRY = retry;  // ❌ 同上
    //     logLevel = level;   // ❌ 编译错误：引用未初始化
    // }

    void showConfig() const {
        cout << "应用名: " << APP_NAME
             << "，最大重试: " << MAX_RETRY
             << "，日志级别: " << logLevel << endl;
    }
};

int main() {
    int level = 3;
    Config cfg("InfiniTensor", 5, level);
    cfg.showConfig();
    // 输出：应用名: InfiniTensor，最大重试: 5，日志级别: 3

    level = 5;  // 修改外部变量
    cfg.showConfig();
    // 输出：日志级别: 5 —— 引用跟着外部变量一起变了！

    return 0;
}
```

> 🎯 **工程规范**：即使对于普通成员变量，也**优先使用初始化列表**。这不仅是性能考量，更是一个明确的信号——「这些成员在对象诞生时应该是什么样」。养成习惯后，你的构造函数会写得又快又安全。

---

## 🔺 第五部分：C++ Big Three（三法则 Rule of Three）

### 5.1 三大法则是什么？

**C++ 三法则（Rule of Three）**是 C++ 中最重要的设计规范之一。它说的是：

> 如果一个类需要自定义**析构函数**、**拷贝构造函数**、**拷贝赋值运算符**中的**任意一个**，那么它几乎一定需要自定义**全部三个**。

为什么是这三个？因为它们都和**资源管理**有关。最常见的触发场景是——**类内部持有指向堆内存的裸指针**。

```cpp
class MyClass {
private:
    int *data;     // ← 裸指针指向堆内存
    // ...
};
// 编译器问：拷贝的时候 data 指针怎么办？
//         析构的时候 data 指向的内存要不要释放？
//         赋值的时候旧内存怎么处理，新内存怎么分配？
// → 这三个问题必须由你来回答，编译器不会替你做对的选择。
```

### 5.2 析构函数（Destructor）

```cpp
~Vector() {
    delete[] data;      // 释放 data 指向的堆内存
    data = nullptr;     // 好习惯：释放后置空
    // 注意：size 和 capacity 等基础类型成员会自动销毁，不需要手动处理
}
```

析构函数的职责非常纯粹：**释放对象持有的所有资源**。对于 Vector，就是 `delete[] data`。如果忘了写析构函数（或者忘了在析构函数里释放内存），每次对象销毁都会泄漏 `capacity * sizeof(T)` 字节的内存。

### 5.3 拷贝构造函数（Copy Constructor）

拷贝构造函数在**用一个已存在的对象来初始化一个新对象**时被调用：

```cpp
// 拷贝构造函数：基于 other 创建一个完全独立的副本
Vector(const Vector &other)
    : size(other.size), capacity(other.capacity) {
    // 分配自己的独立内存空间
    data = new T[capacity];
    // 逐元素拷贝内容
    for (int i = 0; i < size; i++) {
        data[i] = other.data[i];
    }
    cout << "拷贝构造：创建了独立副本（size=" << size << "）" << endl;
}

// 调用场景示例：
Vector<int> a;
a.pushBack(10);
Vector<int> b = a;     // ← 拷贝构造函数在这里被调用
Vector<int> c(a);      // ← 同样是拷贝构造
```

### 5.4 拷贝赋值运算符（Copy Assignment Operator）

拷贝赋值运算符在**两个已经存在的对象之间赋值**时被调用。它的实现比拷贝构造函数多一个关键步骤——**先释放自己的旧资源**：

```cpp
// 拷贝赋值运算符：把 other 的内容拷贝到当前对象
Vector &operator=(const Vector &other) {
    // 步骤 1：防止自我赋值（a = a）
    if (this == &other) {
        return *this;
    }

    // 步骤 2：释放自己的旧资源
    delete[] data;

    // 步骤 3：根据 other 的容量重新分配
    size = other.size;
    capacity = other.capacity;
    data = new T[capacity];

    // 步骤 4：逐元素拷贝
    for (int i = 0; i < size; i++) {
        data[i] = other.data[i];
    }

    cout << "拷贝赋值：完成了深拷贝（size=" << size << "）" << endl;
    return *this;    // 返回自身引用，支持链式赋值 a = b = c
}

// 调用场景示例：
Vector<int> a, b;
a.pushBack(10);
b.pushBack(20);
b = a;               // ← 拷贝赋值运算符在这里被调用
```

> 💡 **为什么 `operator=` 返回 `Vector &`？** 为了支持链式赋值 `a = b = c`。`b = c` 先执行，返回 `b` 的引用，然后 `a = (b的引用)` 再执行。如果返回 `void`，链式赋值就写不了了。

### 5.5 为什么拥有堆内存的类必须遵守三法则

前面 3.2 节我们见识了浅拷贝导致的灾难——两个对象的指针指向同一块内存，析构时发生双重释放。现在我们用 Valgrind 来验证：一个遵守三法则的类 vs 一个不遵守的类，内存表现会差多远。

先看**不遵守三法则**的版本：

```cpp
// bad_vector.cpp —— 只有构造和析构，没有拷贝构造和拷贝赋值
#include <iostream>
using namespace std;

template <typename T>
class BadVector {
public:
    T *data;
    int size;
    int capacity;

    BadVector(int cap = 4) : size(0), capacity(cap) {
        data = new T[capacity];
    }

    ~BadVector() {
        delete[] data;
    }

    void pushBack(const T &val) {
        if (size >= capacity) {
            int newCap = capacity * 2;
            T *newData = new T[newCap];
            for (int i = 0; i < size; i++) newData[i] = data[i];
            delete[] data;
            data = newData;
            capacity = newCap;
        }
        data[size++] = val;
    }

    // ⚠️ 没有拷贝构造函数！编译器生成默认 → 浅拷贝
    // ⚠️ 没有拷贝赋值运算符！编译器生成默认 → 浅拷贝
};

int main() {
    BadVector<int> a;
    a.pushBack(10);
    a.pushBack(20);

    BadVector<int> b = a;   // 浅拷贝！a.data 和 b.data 指向同一块内存
    // main 结束 → a 析构释放内存 → b 析构再次释放同一块 → 双重释放崩溃！
    return 0;
}
```

```bash
# 编译并运行
g++ -g bad_vector.cpp -o bad_vector
./bad_vector
# 大概率输出：
# free(): double free detected in tcache 2
# Aborted (core dumped)
```

再看**遵守三法则**的版本：

```cpp
// good_vector.cpp —— 完整实现三法则
#include <iostream>
using namespace std;

template <typename T>
class GoodVector {
private:
    T *data;
    int size;
    int capacity;

    void resize() {
        int newCap = (capacity == 0) ? 1 : capacity * 2;
        T *newData = new T[newCap];
        for (int i = 0; i < size; i++) newData[i] = data[i];
        delete[] data;
        data = newData;
        capacity = newCap;
    }

public:
    // ① 构造函数
    GoodVector(int cap = 4) : size(0), capacity(cap) {
        data = new T[capacity];
    }

    // ② 析构函数
    ~GoodVector() {
        delete[] data;
        data = nullptr;
    }

    // ③ 拷贝构造函数（深拷贝）
    GoodVector(const GoodVector &other)
        : size(other.size), capacity(other.capacity) {
        data = new T[capacity];
        for (int i = 0; i < size; i++) {
            data[i] = other.data[i];
        }
    }

    // ③ 拷贝赋值运算符（深拷贝）
    GoodVector &operator=(const GoodVector &other) {
        if (this == &other) return *this;
        delete[] data;
        size = other.size;
        capacity = other.capacity;
        data = new T[capacity];
        for (int i = 0; i < size; i++) {
            data[i] = other.data[i];
        }
        return *this;
    }

    void pushBack(const T &val) {
        if (size >= capacity) resize();
        data[size++] = val;
    }

    T &operator[](int i) { return data[i]; }
    int getSize() const { return size; }
};

int main() {
    GoodVector<int> a;
    a.pushBack(10);
    a.pushBack(20);

    GoodVector<int> b = a;    // ✅ 深拷贝：b 有自己独立的内存
    GoodVector<int> c;
    c = a;                    // ✅ 深拷贝：c 也有自己独立的内存

    a[0] = 999;               // 修改 a 不影响 b 和 c
    cout << "a[0] = " << a[0] << endl;   // 999
    cout << "b[0] = " << b[0] << endl;   // 10（独立副本）
    cout << "c[0] = " << c[0] << endl;   // 10（独立副本）

    // main 结束 → a、b、c 分别析构 → 各自释放各自的内存 ✅
    return 0;
}
```

```bash
# 编译运行并 Valgrind 验证
g++ -g good_vector.cpp -o good_vector
valgrind --leak-check=full ./good_vector
# 输出：All heap blocks were freed -- no leaks are possible
```

### 5.6 不遵守三法则引发的内存泄漏与重复释放实战总结

| 缺失哪个 | 后果 | 严重程度 |
|---------|------|---------|
| **缺失析构函数** | `new[]` 的内存永远不会 `delete[]` → **内存泄漏** | ⭐⭐⭐⭐⭐ 长期运行必出事 |
| **缺失拷贝构造** | `Vector b = a;` 时两个对象共享同一块堆内存 → 析构时**双重释放** | ⭐⭐⭐⭐⭐ 直接崩溃 |
| **缺失拷贝赋值** | `b = a;` 时旧内存没释放 → **内存泄漏** + 共享同一块堆内存 → **双重释放** | ⭐⭐⭐⭐⭐ 泄漏+崩溃二合一 |

> 🎯 **记忆口诀**：「有堆指针，三法全写」。析构释放、拷贝构造独立分配、拷贝赋值先释放再分配。少一个都不行。

---

## ⚡ 第六部分：移动语义与 C++ Big Five（五法则 Rule of Five）

### 6.1 C++11 移动语义的设计思想

拷贝操作有一个固有效率的瓶颈——**数据是「复制」的，而非「转移」的**。想象一个场景：

```cpp
Vector<int> createLargeVector() {
    Vector<int> temp;
    for (int i = 0; i < 1000000; i++) {
        temp.pushBack(i);    // 往 temp 里加了 100 万个元素
    }
    return temp;   // temp 马上就要销毁了，但 return 时还要把 100 万个元素拷贝一份……
}

int main() {
    Vector<int> v = createLargeVector();
    // 如果不支持移动语义：
    //   1. temp 创建，堆上分配 ~4MB
    //   2. return 时拷贝构造 v，堆上再分配 ~4MB，逐个拷贝 100 万个元素
    //   3. temp 析构，释放第一块 ~4MB
    //   → 峰值内存 8MB + 100 万次拷贝，全是浪费！
    return 0;
}
```

C++ 的设计者们看到了这个痛点。temp 马上就要销毁了，**为什么要把它的数据「复制」一份？直接把 temp 的指针「偷」过来不行吗？**

这就是**移动语义（Move Semantics）**的核心思想：对于即将销毁的临时对象，不复制它的资源，而是**转移所有权**——把它的堆内存指针直接「接管」过来，省去分配和拷贝的开销。

```
拷贝 vs 移动的直观对比（100 万个元素的 Vector）：

拷贝构造：
  temp.data ──→ [0][1][2]...[999999]    temp 的堆内存
  v.data    ──→ [0][1][2]...[999999]    v 的堆内存（全新分配，逐元素复制）
  → 时间：O(n) 拷贝 + 额外内存分配
  → 空间：峰值 2 倍内存

移动构造：
  temp.data ──→ [0][1][2]...[999999]    ← v.data 直接指向这里！
  v.data    ──→ [0][1][2]...[999999]    
  temp.data = nullptr                    ← temp 放弃了所有权
  → 时间：O(1)（只改了 3 个指针/整数）
  → 空间：始终只有 1 份内存
```

### 6.2 移动构造函数与移动赋值函数

C++11 引入了**右值引用 `&&`** 来标识「可以被移动的临时对象」。移动构造函数和移动赋值运算符就使用这个语法：

```cpp
// ===== 移动构造函数 =====
// 参数是 T&& —— 右值引用，表示「我要接管你的资源」
Vector(Vector &&other) noexcept
    // : data(nullptr), size(0), capacity(0)    // 也可以先初始化
    : data(other.data)           // 直接把 other 的指针「偷」过来
    , size(other.size)           // 把 other 的 size 也偷过来
    , capacity(other.capacity) { // 把 other 的 capacity 也偷过来

    // 关键步骤：把 other 掏空，让它处于「可以被安全析构」的状态
    other.data = nullptr;
    other.size = 0;
    other.capacity = 0;

    cout << "移动构造：资源已转移（size=" << size << "）" << endl;
}

// ===== 移动赋值运算符 =====
Vector &operator=(Vector &&other) noexcept {
    // 步骤 1：防止自我赋值（虽然对于移动来说很罕见，但还是要检查）
    if (this == &other) {
        return *this;
    }

    // 步骤 2：释放自己的旧资源
    delete[] data;

    // 步骤 3：「偷」走 other 的资源
    data = other.data;
    size = other.size;
    capacity = other.capacity;

    // 步骤 4：把 other 掏空
    other.data = nullptr;
    other.size = 0;
    other.capacity = 0;

    cout << "移动赋值：资源已转移（size=" << size << "）" << endl;
    return *this;
}
```

> 💡 **`noexcept` 关键字**：移动构造函数和移动赋值运算符**强烈建议**标记为 `noexcept`。这不仅告诉编译器「这个函数不会抛异常」，还有一个非常实际的工程收益——`std::vector` 在扩容时，只有元素类型的移动构造函数是 `noexcept` 的，它才会用移动而非拷贝。如果没标 `noexcept`，`std::vector` 为了「安全保证」会退化为拷贝——之前的性能优化全白费了。

### 6.3 拷贝与移动的性能差异实测

来写一个简单的时间对比，直观感受移动语义的威力：

```cpp
#include <iostream>
#include <chrono>
using namespace std;
using namespace chrono;

// ===== 一个大对象（模拟张量数据） =====
class LargeData {
private:
    int *data;
    int size;
    static int copyCount;
    static int moveCount;

public:
    LargeData(int s = 10000000) : size(s) {   // 1000 万个 int ≈ 40MB
        data = new int[size];
    }

    // 拷贝构造（昂贵的操作）
    LargeData(const LargeData &other) : size(other.size) {
        data = new int[size];
        for (int i = 0; i < size; i++) {
            data[i] = other.data[i];      // 逐个拷贝 1000 万个元素
        }
        copyCount++;
    }

    // 移动构造（廉价的操作）
    LargeData(LargeData &&other) noexcept
        : data(other.data), size(other.size) {
        other.data = nullptr;             // 只改了 2 个指针/整数
        other.size = 0;
        moveCount++;
    }

    ~LargeData() { delete[] data; }

    static void resetCount() { copyCount = moveCount = 0; }
    static void printCount() {
        cout << "  拷贝次数: " << copyCount
             << "，移动次数: " << moveCount << endl;
    }
};

int LargeData::copyCount = 0;
int LargeData::moveCount = 0;

// ===== 对比测试 =====
LargeData createData() {
    LargeData temp;
    return temp;   // C++11 起，return 局部变量会自动使用移动语义
}

int main() {
    cout << "===== 移动语义性能对比 =====" << endl;

    // 测试 1：移动构造
    LargeData::resetCount();
    auto start = high_resolution_clock::now();
    LargeData d1 = createData();   // 触发移动构造（或 RVO 优化）
    auto end = high_resolution_clock::now();
    auto duration = duration_cast<milliseconds>(end - start).count();
    cout << "移动构造耗时: " << duration << " ms" << endl;
    LargeData::printCount();

    // 测试 2：强制拷贝构造（使用 std::move 的反面——直接拷贝）
    LargeData::resetCount();
    LargeData original;
    start = high_resolution_clock::now();
    LargeData d2 = original;       // 触发拷贝构造，逐个拷贝 1000 万个元素
    end = high_resolution_clock::now();
    duration = duration_cast<milliseconds>(end - start).count();
    cout << "拷贝构造耗时: " << duration << " ms" << endl;
    LargeData::printCount();

    return 0;
}
```

```bash
g++ -O2 move_bench.cpp -o move_bench
./move_bench
```

典型输出（具体数值取决于机器性能）：

```
===== 移动语义性能对比 =====
移动构造耗时: 0 ms
  拷贝次数: 0，移动次数: 1
拷贝构造耗时: 85 ms
  拷贝次数: 1，移动次数: 0
```

**移动构造几乎是零开销完成 40MB 数据的转移，而拷贝构造需要 85ms**。在大模型推理中，一个张量可能在多个算子之间传递——如果没有移动语义，每次传参都是一次完整的内存拷贝，累积下来性能损失惊人。

### 6.4 现代 C++ 工程五法则完整规范

C++11 引入了移动语义之后，三法则扩展成了**五法则（Rule of Five）**：

> 如果一个类需要自定义**析构函数、拷贝构造函数、拷贝赋值运算符、移动构造函数、移动赋值运算符**中的**任意一个**，那么它通常需要自定义**全部五个**。

但实践中有个更现代的说法——**Rule of Zero（零法则）**：

> 如果你能用 `std::vector`、`std::string`、`std::unique_ptr` 等 STL 组件来管理资源，那么**一个都不用写**——编译器生成的默认版本就是正确的。

对于我们手写的 Vector 类（持有裸指针），完整五法则的实现如下：

```cpp
// Vector 五法则完整声明
template <typename T>
class Vector {
private:
    T *data;
    int size;
    int capacity;

public:
    // ① 构造函数
    Vector(int cap = 4);

    // ② 析构函数
    ~Vector();

    // ③ 拷贝构造函数
    Vector(const Vector &other);

    // ④ 拷贝赋值运算符
    Vector &operator=(const Vector &other);

    // ⑤ 移动构造函数（C++11）
    Vector(Vector &&other) noexcept;

    // ⑥ 移动赋值运算符（C++11）
    Vector &operator=(Vector &&other) noexcept;
};
```

| 法则名称 | 包含哪些函数 | 适用场景 |
|---------|------------|---------|
| **Rule of Three** | 析构 + 拷贝构造 + 拷贝赋值 | C++98/03 时代，管理堆资源的类 |
| **Rule of Five** | 析构 + 拷贝构造 + 拷贝赋值 + 移动构造 + 移动赋值 | C++11+ 现代工程，全面支持移动语义 |
| **Rule of Zero** | 一个都不写 | 用 STL 组件管理资源，默认生成的就是对的 |

> 🎯 **现代 C++ 最佳实践**：优先追求 **Rule of Zero**——用 `std::vector` 代替裸 `new[]`，用 `std::unique_ptr` 代替裸指针。只有当你有充分理由自己管理资源时，才退回到 Rule of Five。这里我们手写 Vector 只是为了**深入理解底层原理**——理解之后，工业代码直接用 `std::vector` 就好。

---

## 🗄️ 第七部分：STL 标准容器体系与特性详解

### 7.1 常用 STL 容器一览

C++ **标准模板库（STL, Standard Template Library）**提供了一套久经考验的通用数据结构和算法。我们手写 Vector 踩过的所有坑——扩容策略、深拷贝、移动语义、迭代器……STL 都已经帮你完美解决了。下面是五种最常用的 STL 容器：

```cpp
#include <vector>         // 动态数组
#include <deque>          // 双端队列
#include <list>           // 双向链表
#include <map>            // 有序键值对（红黑树）
#include <unordered_map>  // 哈希表
```

### 7.2 各容器底层结构、存取效率与优缺点

| 容器 | 底层结构 | 随机访问 | 头/尾插入 | 中间插入/删除 | 查找 | 内存特点 |
|------|---------|---------|----------|-------------|------|---------|
| **`vector`** | 连续数组 | O(1) | O(1) 尾插 | O(n)（需移动元素） | O(n) | 内存连续，缓存友好 |
| **`deque`** | 分段连续数组 | O(1) | O(1) 头尾均可 | O(n) | O(n) | 支持高效头插，内存略分散 |
| **`list`** | 双向链表 | 不支持（只能顺序遍历） | O(1) | O(1)（已知位置时） | O(n) | 每个节点有前驱后继指针，内存碎片化 |
| **`map`** | 红黑树 | 不支持 | O(log n) | O(log n) | O(log n) | 自动按键排序，键唯一 |
| **`unordered_map`** | 哈希表 | 不支持 | O(1) 均摊 | O(1) 均摊 | O(1) 均摊 | 无序，哈希冲突时退化为 O(n) |

用代码来直观对比各容器的基础用法：

```cpp
#include <iostream>
#include <vector>
#include <deque>
#include <list>
#include <map>
#include <unordered_map>
using namespace std;

int main() {
    // ===== vector：最常用的动态数组 =====
    vector<int> vec = {10, 20, 30};
    vec.push_back(40);            // 尾插：O(1)
    vec[0] = 100;                 // 随机访问：O(1)
    cout << "vec[2] = " << vec[2] << endl;   // 30

    // ===== deque：双端队列，头尾都能高效插入 =====
    deque<int> dq = {10, 20, 30};
    dq.push_front(5);             // 头插：O(1)，这是 vector 做不到的
    dq.push_back(40);             // 尾插：O(1)
    cout << "dq front: " << dq.front() << ", back: " << dq.back() << endl;
    // 输出：dq front: 5, back: 40

    // ===== list：双向链表，任意位置插入删除都是 O(1) =====
    list<int> lst = {10, 20, 30};
    auto it = lst.begin();
    ++it;                          // 指向第二个元素 20
    lst.insert(it, 15);            // 在 20 前面插入 15：O(1)
    for (int x : lst) cout << x << " ";   // 输出：10 15 20 30
    cout << endl;

    // ===== map：有序键值对（红黑树实现） =====
    map<string, int> scores;
    scores["张三"] = 88;
    scores["李四"] = 93;
    scores["王五"] = 76;
    // 遍历时自动按键（string 字典序）排序输出
    cout << "=== 成绩表（按姓名排序）===" << endl;
    for (const auto &[name, score] : scores) {
        cout << name << ": " << score << endl;
    }
    // 输出：李四: 93  王五: 76  张三: 88（自动按拼音排序）

    // ===== unordered_map：哈希表，O(1) 查找 =====
    unordered_map<string, int> hashScores;
    hashScores["张三"] = 88;
    hashScores["李四"] = 93;
    hashScores["王五"] = 76;
    // 查找指定键：O(1)
    if (hashScores.find("李四") != hashScores.end()) {
        cout << "李四的成绩: " << hashScores["李四"] << endl;  // 93
    }

    return 0;
}
```

### 7.3 AI 开发、张量计算场景下的容器选型原则

在 InfiniTensor 等大模型框架中，面对不同的数据结构需求，选对容器对性能影响巨大：

| 场景 | 推荐容器 | 原因 |
|------|---------|------|
| **张量数据存储** | `std::vector` | 连续内存、缓存友好、支持 `data()` 获取裸指针传给 CUDA/cuBLAS |
| **模型权重字典** | `std::unordered_map<string, Tensor>` | O(1) 按名称查找权重，不需要排序 |
| **算子注册表** | `std::map<string, OpCreator>` | 需要有序，便于调试和遍历 |
| **计算图节点** | `std::list` 或自定义结构 | 频繁的插入删除，不需要随机访问 |
| **推理请求队列** | `std::deque` | 支持高效的头出队、尾入队 |
| **KV Cache** | `std::vector`（预分配大块） | 连续内存，减少碎片，方便 GPU 传输 |

> 🎯 **选型三问**：①需要随机访问吗？→ `vector` / `deque`；②需要频繁中间插入删除吗？→ `list`；③需要按键快速查找吗？→ `unordered_map`（无序快查）或 `map`（有序慢查）。

---

## 🧬 第八部分：面向对象高阶——继承、多态、抽象类

### 8.1 类的继承机制与访问权限控制

**继承（Inheritance）**是面向对象编程的三大特性之一（另外两个是封装和多态）。它允许一个类（派生类/子类）从另一个类（基类/父类）**继承**成员变量和成员函数，实现代码复用和「is-a」的语义关系。

```cpp
#include <iostream>
#include <string>
using namespace std;

// ===== 基类：算子的通用接口 =====
class Operator {
protected:                   // protected：派生类可以访问，外部不能访问
    string opName;
    int inputCount;

public:
    Operator(string name, int inputs)
        : opName(name), inputCount(inputs) {
        cout << "基类 Operator 构造：" << opName << endl;
    }

    virtual ~Operator() {    // ⚠️ 虚析构函数——马上在多态部分详细解释
        cout << "基类 Operator 析构：" << opName << endl;
    }

    void showInfo() const {
        cout << "算子：" << opName
             << "，输入数：" << inputCount << endl;
    }
};

// ===== 派生类：矩阵乘法算子 =====
// public 继承：基类的 public 成员在派生类中仍是 public
class MatMulOp : public Operator {
private:
    int m, n, k;   // 矩阵维度：A[m×k] × B[k×n]

public:
    MatMulOp(int m_, int n_, int k_)
        : Operator("MatMul", 2)   // 先调用基类构造函数
        , m(m_), n(n_), k(k_) {
        cout << "派生类 MatMulOp 构造："
             << m << "×" << k << " × " << k << "×" << n << endl;
    }

    ~MatMulOp() {
        cout << "派生类 MatMulOp 析构" << endl;
    }

    void compute() const {
        cout << "执行 " << opName    // opName 是 protected，派生类可以访问
             << "：(" << m << "×" << k << ") × ("
             << k << "×" << n << ") → (" << m << "×" << n << ")" << endl;
    }
};

int main() {
    MatMulOp mm(512, 512, 512);
    mm.showInfo();     // 继承了基类的 showInfo()
    mm.compute();      // 派生类自己的方法

    // 构造顺序：基类构造 → 派生类构造
    // 析构顺序：派生类析构 → 基类析构（与构造顺序相反）
    return 0;
}
```

运行输出：

```
基类 Operator 构造：MatMul
派生类 MatMulOp 构造：512×512 × 512×512
算子：MatMul，输入数：2
执行 MatMul：(512×512) × (512×512) → (512×512)
派生类 MatMulOp 析构
基类 Operator 析构：MatMul
```

三种继承方式的访问权限变化：

| 基类成员权限 | **`public` 继承** | **`protected` 继承** | **`private` 继承** |
|------------|------------------|---------------------|-------------------|
| `public` | → `public` ✅ 最常用 | → `protected` | → `private` |
| `protected` | → `protected` | → `protected` | → `private` |
| `private` | 不可访问 | 不可访问 | 不可访问 |

> 🎯 **工程上 95% 的情况都用 `public` 继承**。`protected` 和 `private` 继承更多用于「用某个类来实现我」而非「我是一个……」的场景——属于进阶技巧，初学时了解即可。

### 8.2 多态的实现原理与虚函数作用

**多态（Polymorphism）**是面向对象编程的灵魂。它让你可以用**基类的指针或引用**来操作**派生类的对象**，程序在运行时自动选择正确的函数版本——这就是「同一接口，不同行为」。

```cpp
#include <iostream>
#include <string>
using namespace std;

// ===== 基类：算子 =====
class Operator {
protected:
    string opName;

public:
    Operator(string name) : opName(name) {}

    // ===== 虚函数：核心！ =====
    // virtual 告诉编译器：「派生类可能会覆盖这个函数，
    // 请通过虚函数表（vtable）在运行时查找正确的版本」
    virtual void execute() const {
        cout << "Operator::execute() —— 基类默认实现" << endl;
    }

    // ⚠️ 虚析构函数：如果基类指针指向派生类对象，
    // 没有虚析构函数会导致派生类的析构函数不被调用 → 内存泄漏！
    virtual ~Operator() {
        cout << "~Operator()" << endl;
    }
};

// ===== 派生类：MatMul =====
class MatMul : public Operator {
public:
    MatMul() : Operator("MatMul") {}

    // override 关键字：明确告诉编译器「我在覆盖基类的虚函数」
    // 如果写错了函数签名，编译器会报错——这是一个非常实用的安全检查
    void execute() const override {
        cout << "MatMul::execute() —— 执行矩阵乘法" << endl;
    }
};

// ===== 派生类：ReLU =====
class ReLU : public Operator {
public:
    ReLU() : Operator("ReLU") {}

    void execute() const override {
        cout << "ReLU::execute() —— 执行逐元素激活" << endl;
    }
};

// ===== 派生类：Softmax =====
class Softmax : public Operator {
public:
    Softmax() : Operator("Softmax") {}

    void execute() const override {
        cout << "Softmax::execute() —— 执行 softmax 归一化" << endl;
    }
};

// ===== 统一的执行接口：不关心具体是哪种算子！ =====
void runOperator(const Operator &op) {
    op.execute();   // 运行时根据实际类型选择正确的 execute()
}

int main() {
    MatMul matmul;
    ReLU relu;
    Softmax softmax;

    // 用基类引用操作三种不同的派生类对象
    // 调用的是各自的 execute()，不是基类的 —— 这就是多态！
    runOperator(matmul);    // 输出：MatMul::execute() —— 执行矩阵乘法
    runOperator(relu);      // 输出：ReLU::execute() —— 执行逐元素激活
    runOperator(softmax);   // 输出：Softmax::execute() —— 执行 softmax 归一化

    // ===== 多态的经典用法：基类指针数组 =====
    cout << "\n=== 统一调度所有算子 ===" << endl;
    Operator *pipeline[] = {&matmul, &relu, &softmax};
    for (Operator *op : pipeline) {
        op->execute();     // 同一行代码，行为各不相同
    }

    return 0;
}
```

**多态的实现原理——虚函数表（vtable）**：

```
每个包含虚函数的类，编译器会为它生成一张「虚函数表」（vtable），
里面存着所有虚函数的实际地址。

   Operator 的 vtable:              MatMul 的 vtable:
   ┌──────────────────┐            ┌──────────────────┐
   │ execute → Operator::execute   │ execute → MatMul::execute
   │ ~Operator → Operator::~Op     │ ~Operator → MatMul::~MatMul
   └──────────────────┘            └──────────────────┘

当通过 Operator* 调用 op->execute() 时：
  1. 找到 op 指向的对象的 vtable（运行时的实际类型决定是哪张表）
  2. 从 vtable 中找到 execute 的实际地址
  3. 调用那个地址 → 实现了「同一调用，不同行为」
```

> ⚠️ **虚析构函数的必要性**：如果代码中有 `Operator *p = new MatMul(); delete p;`，但 `~Operator()` 不是虚函数，那么 `delete p` 只会调用 `~Operator()`，`MatMul` 特有的资源（如果有的话）不会被释放——导致**内存泄漏**。**基类的析构函数要么写 `virtual ~Base()`，要么用 `protected` 非虚析构函数（禁止通过基类指针 delete）——二选一，不能裸奔。**

### 8.3 抽象类：定义「必须实现什么」，不关心「怎么实现」

**抽象类（Abstract Class）**是不能被直接实例化的类，它包含至少一个**纯虚函数**。它的作用是定义一套**接口规范**——告诉所有派生类「你必须实现这些方法」，至于怎么实现，每个派生类自己决定。

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
using namespace std;

// ===== 抽象基类：定义「层（Layer）」的统一接口 =====
class Layer {
protected:
    string layerName;

public:
    Layer(string name) : layerName(name) {}
    virtual ~Layer() {}

    // ===== 纯虚函数（= 0）：派生类必须实现！ =====
    virtual void forward() = 0;     // 前向传播
    virtual void backward() = 0;    // 反向传播
    virtual int paramCount() const = 0;  // 参数量

    // 普通虚函数：可以有默认实现，派生类可以覆盖也可以不覆盖
    virtual string getLayerName() const {
        return layerName;
    }
};

// ===== 具体派生类：全连接层 =====
class LinearLayer : public Layer {
private:
    int inFeatures, outFeatures;

public:
    LinearLayer(int inF, int outF)
        : Layer("Linear"), inFeatures(inF), outFeatures(outF) {}

    void forward() override {
        cout << "[Linear] 前向传播："
             << inFeatures << " → " << outFeatures << endl;
    }

    void backward() override {
        cout << "[Linear] 反向传播：计算梯度" << endl;
    }

    int paramCount() const override {
        return inFeatures * outFeatures + outFeatures;  // W + b
    }
};

// ===== 具体派生类：卷积层 =====
class ConvLayer : public Layer {
private:
    int inCh, outCh, kernelSize;

public:
    ConvLayer(int inC, int outC, int k)
        : Layer("Conv2D"), inCh(inC), outCh(outC), kernelSize(k) {}

    void forward() override {
        cout << "[Conv2D] 前向传播：" << inCh
             << " → " << outCh << " channels, kernel=" << kernelSize << endl;
    }

    void backward() override {
        cout << "[Conv2D] 反向传播：计算卷积梯度" << endl;
    }

    int paramCount() const override {
        return outCh * inCh * kernelSize * kernelSize + outCh;  // W + b
    }
};

int main() {
    // Layer base;  // ❌ 编译错误！抽象类不能直接实例化

    // ✅ 通过基类指针使用派生类对象
    vector<unique_ptr<Layer>> model;
    model.push_back(make_unique<LinearLayer>(784, 256));
    model.push_back(make_unique<ConvLayer>(3, 64, 3));
    model.push_back(make_unique<LinearLayer>(256, 10));

    // ===== 统一的训练循环：不关心每层具体是什么 =====
    cout << "========== 模型结构 ==========" << endl;
    int totalParams = 0;
    for (const auto &layer : model) {
        cout << layer->getLayerName() << "："
             << layer->paramCount() << " 个参数" << endl;
        totalParams += layer->paramCount();
    }
    cout << "总参数量：" << totalParams << endl;

    cout << "\n========== 前向传播 ==========" << endl;
    for (const auto &layer : model) {
        layer->forward();    // 多态：每层各做各的 forward
    }

    cout << "\n========== 反向传播 ==========" << endl;
    for (auto it = model.rbegin(); it != model.rend(); ++it) {
        (*it)->backward();   // 多态：每层各做各的 backward
    }

    return 0;
}
```

运行输出：

```
========== 模型结构 ==========
Linear：200960 个参数
Conv2D：1792 个参数
Linear：2570 个参数
总参数量：205322

========== 前向传播 ==========
[Linear] 前向传播：784 → 256
[Conv2D] 前向传播：3 → 64 channels, kernel=3
[Linear] 前向传播：256 → 10

========== 反向传播 ==========
[Linear] 反向传播：计算梯度
[Conv2D] 反向传播：计算卷积梯度
[Linear] 反向传播：计算梯度
```

> 🎯 **抽象类在 AI 框架中的应用**：InfiniTensor 中有大量的抽象接口设计——`Operator`（算子基类）、`Allocator`（内存分配器接口）、`Device`（设备抽象）、`Tensor`（张量基类）。每一种具体实现（CUDA 算子、CPU 算子、Vulkan 算子）都继承自这些抽象接口，框架的上层代码只依赖抽象接口，完全不关心底层是哪种硬件——这正是**依赖倒置原则**在大型 C++ 项目中的经典实践。

---

## 🧠 第九部分：智能指针基础

### 9.1 裸指针的内存管理痛点

我们已经用 `new` / `delete` 管理了好几篇文章的堆内存。裸指针（raw pointer）虽然灵活，但在复杂工程中有三个致命缺陷：

```cpp
// 痛点 1：容易忘记 delete → 内存泄漏
void processData() {
    int *buf = new int[1024];
    // ... 处理数据 ...
    if (someCondition) {
        return;              // ❌ 提前返回了，忘记 delete[] buf → 泄漏！
    }
    // ... 更多处理 ...
    delete[] buf;            // 正常情况下能走到这里
}

// 痛点 2：异常安全 —— 抛出异常时 delete 被跳过
void riskyOperation() {
    int *data = new int[1000000];
    doSomething();           // 如果这里抛出了异常……
    delete[] data;           // 这一行永远不会执行！→ 泄漏
}

// 痛点 3：所有权不清晰 —— 谁负责 delete？
int *createBuffer() {
    return new int[1024];    // 把这个指针返回给调用方了
}                            // 调用方知道要 delete[] 吗？文档里写了吗？
                             // 团队协作中这几乎一定会出错
```

### 9.2 unique_ptr：独占所有权，自动释放

**`std::unique_ptr`** 是最轻量级的智能指针。它**独占**指向的对象，不能被拷贝（只能移动），离开作用域时自动 `delete`。

```cpp
#include <iostream>
#include <memory>          // 智能指针都在这个头文件里
using namespace std;

class Tensor {
private:
    double *data;
    int size;

public:
    Tensor(int s) : size(s) {
        data = new double[size];
        cout << "Tensor 构造：分配了 " << size * sizeof(double) << " 字节" << endl;
    }

    ~Tensor() {
        delete[] data;
        cout << "Tensor 析构：释放了 " << size * sizeof(double) << " 字节" << endl;
    }

    int getSize() const { return size; }
};

// ===== 工厂函数：返回 unique_ptr，所有权清晰明确 =====
unique_ptr<Tensor> createTensor(int size) {
    // make_unique：安全地创建 unique_ptr（C++14 起）
    return make_unique<Tensor>(size);
    // 等价于（C++11）：return unique_ptr<Tensor>(new Tensor(size));
}

int main() {
    cout << "===== unique_ptr 基础用法 =====" << endl;

    // ① 创建 unique_ptr
    unique_ptr<Tensor> t1 = make_unique<Tensor>(1000);

    // ② 像普通指针一样使用（-> 和 *）
    cout << "Tensor 大小：" << t1->getSize() << endl;

    // ③ unique_ptr 不能被拷贝（独占所有权）
    // unique_ptr<Tensor> t2 = t1;   // ❌ 编译错误！拷贝构造被 = delete 了

    // ④ 但可以被移动：所有权从 t1 转移给 t3
    unique_ptr<Tensor> t3 = move(t1);   // ✅ t1 变成 nullptr，t3 拥有对象
    if (t1 == nullptr) {
        cout << "t1 已空（所有权已转给 t3）" << endl;
    }

    // ⑤ 离开作用域 → t3 自动 delete → Tensor 析构函数被调用 ✅
    // 没有任何显式的 delete！也不需要 try-catch 来保证释放！

    cout << "\n===== unique_ptr 防止内存泄漏 =====" << endl;

    {
        auto t = make_unique<Tensor>(500);
        cout << "进入作用域，创建了 Tensor" << endl;
        // 即使这里 return 或抛出异常……
        cout << "离开作用域——" << endl;
    }   // ← 无论怎样退出作用域，t 都会自动释放内存
    cout << "已离开作用域，Tensor 已自动释放 ✅" << endl;

    return 0;
}
```

运行输出：

```
===== unique_ptr 基础用法 =====
Tensor 构造：分配了 8000 字节
Tensor 大小：1000
t1 已空（所有权已转给 t3）
Tensor 析构：释放了 8000 字节

===== unique_ptr 防止内存泄漏 =====
Tensor 构造：分配了 4000 字节
进入作用域，创建了 Tensor
离开作用域——
Tensor 析构：释放了 4000 字节
已离开作用域，Tensor 已自动释放 ✅
```

### 9.3 shared_ptr：共享所有权，引用计数

**`std::shared_ptr`** 允许多个智能指针**共享**同一个对象。内部维护一个**引用计数**——每多一个 `shared_ptr` 指向同一个对象，计数 +1；每销毁一个，计数 -1。当计数归零时，自动释放对象。

```cpp
#include <iostream>
#include <memory>
#include <string>
using namespace std;

class Model {
private:
    string name;
    int paramCount;

public:
    Model(string n, int p) : name(n), paramCount(p) {
        cout << "Model 构造：" << name << "（" << paramCount << " 参数）" << endl;
    }

    ~Model() {
        cout << "Model 析构：" << name << endl;
    }

    void describe() const {
        cout << "模型 " << name << "，参数量：" << paramCount << endl;
    }
};

int main() {
    cout << "===== shared_ptr 基础用法 =====" << endl;

    // ① 创建 shared_ptr
    auto model1 = make_shared<Model>("GPT-mini", 1000000);
    cout << "引用计数：" << model1.use_count() << endl;   // 1

    {
        // ② 拷贝 shared_ptr：引用计数 +1，不拷贝底层对象
        shared_ptr<Model> model2 = model1;
        cout << "拷贝后引用计数：" << model1.use_count() << endl;  // 2

        shared_ptr<Model> model3 = model1;
        cout << "再拷贝后引用计数：" << model1.use_count() << endl;  // 3

        // 三个指针指向同一个 Model 对象
        model1->describe();
        model2->describe();
        model3->describe();

        // model2 和 model3 离开作用域 → 引用计数 -2 → 变成 1
        cout << "model2、model3 即将离开作用域……" << endl;
    }

    // ③ 引用计数回到 1，对象还活着
    cout << "内层作用域结束后，引用计数：" << model1.use_count() << endl;  // 1
    model1->describe();   // 仍然可以使用

    // ④ model1 离开 main → 引用计数 -1 → 变成 0 → Model 析构 ✅
    cout << "main 即将结束……" << endl;
    return 0;
}
```

运行输出：

```
===== shared_ptr 基础用法 =====
Model 构造：GPT-mini（1000000 参数）
引用计数：1
拷贝后引用计数：2
再拷贝后引用计数：3
模型 GPT-mini，参数量：1000000
模型 GPT-mini，参数量：1000000
模型 GPT-mini，参数量：1000000
model2、model3 即将离开作用域……
内层作用域结束后，引用计数：1
模型 GPT-mini，参数量：1000000
main 即将结束……
Model 析构：GPT-mini
```

### 9.4 智能指针如何杜绝内存泄漏——对比总结

| 特性 | **裸指针 `T*`** | **`unique_ptr<T>`** | **`shared_ptr<T>`** |
|------|----------------|-------------------|-------------------|
| 自动释放 | ❌ 必须手动 `delete` | ✅ 离开作用域自动释放 | ✅ 引用计数归零自动释放 |
| 所有权 | 不明确 | 独占（不可拷贝，可移动） | 共享（引用计数） |
| 拷贝 | 浅拷贝（危险） | 禁止拷贝 | 拷贝后计数 +1（安全） |
| 性能开销 | 零开销（只是地址） | 几乎零开销 | 引用计数的原子操作有少量开销 |
| 使用场景 | 非拥有型观察、与 C API 交互 | 工厂函数返回值、容器中的对象 | 多个对象共享同一资源、缓存 |

> 🎯 **现代 C++ 的指针使用原则**：① 能用 `unique_ptr` 就不用手动 `new`/`delete`；② 需要共享所有权时用 `shared_ptr`；③ 裸指针只用于「观察」（不拥有所有权），意味着你不需要对它负责释放。做到这三点，你就能和内存泄漏彻底说再见。

---

## 🔑 第十部分：现代 C++ 关键字——auto 与 using

### 10.1 auto：自动类型推导

**`auto`** 让编译器根据初始化表达式自动推导变量的类型。它不是「弱类型」——C++ 仍然是强类型语言，只是你不用手写繁琐的类型名了。

```cpp
#include <iostream>
#include <vector>
#include <map>
#include <memory>
using namespace std;

int main() {
    // ===== 场景 1：基础类型（意义不大，但完全可以） =====
    auto x = 42;              // int
    auto pi = 3.14159;        // double
    auto name = "InfiniTensor";  // const char*

    // ===== 场景 2：迭代器——auto 的最大受益者 =====
    vector<int> vec = {10, 20, 30, 40, 50};

    // ❌ 不写 auto 的话，迭代器类型又臭又长：
    // vector<int>::iterator it = vec.begin();
    // 或者更长的：
    // vector<pair<string, vector<double>>>::const_iterator it = ...;

    // ✅ 用 auto：类型从 vec.begin() 的返回值自动推导
    for (auto it = vec.begin(); it != vec.end(); ++it) {
        cout << *it << " ";
    }
    cout << endl;

    // ===== 场景 3：范围 for 循环（C++11 起） =====
    for (auto val : vec) {         // val 是 int（拷贝）
        cout << val << " ";
    }
    cout << endl;

    for (auto &val : vec) {        // val 是 int&（引用，可以修改原值）
        val *= 2;
    }
    // vec 变成了 {20, 40, 60, 80, 100}

    for (const auto &val : vec) {  // val 是 const int&（只读引用，高效）
        cout << val << " ";
    }
    cout << endl;

    // ===== 场景 4：简化模板和智能指针类型 =====
    auto ptr = make_unique<vector<double>>();  // unique_ptr<vector<double>>
    auto sp = make_shared<map<string, int>>(); // shared_ptr<map<string, int>>

    // ===== 场景 5：结构化绑定 + auto（C++17 起） =====
    map<string, int> scores = {{"张三", 88}, {"李四", 93}};
    for (const auto &[name, score] : scores) {   // 结构化绑定
        cout << name << ": " << score << endl;
    }

    return 0;
}
```

**auto 使用建议**：

| 建议 | 说明 |
|------|------|
| ✅ 迭代器、智能指针、模板类型 | `auto` 大幅提升可读性，这就是它的主场 |
| ✅ 范围 for 循环 | `for (auto &x : container)` 已经成为 C++ 的惯用写法 |
| ⚠️ 基础类型 `int`、`double` | 影响不大，写不写 `auto` 都可以，团队统一风格即可 |
| ❌ 如果类型不明显、需要靠推导来理解 | 比如 `auto result = someObscureFunction();` —— 读者不知道 `result` 是什么类型，可读性反而降低 |

### 10.2 using：类型别名与现代命名空间

**`using`** 有两个完全不同的用途，需要分清楚：

#### 用途一：类型别名（替代 C 语言的 `typedef`）

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <unordered_map>
#include <string>
using namespace std;

// ===== typedef 语法（C 语言风格，可读性差） =====
typedef vector<int> IntVec;                           // int 的 vector
typedef unordered_map<string, int> StringIntMap;      // string → int 的 map
typedef void (*FuncPtr)(int, double);                 // 函数指针……typedef 语法极不直观

// ===== using 语法（现代 C++ 风格，可读性远胜 typedef） =====
using IntVector = vector<int>;                        // 清晰：把 A 叫做 B
using StringIntMap = unordered_map<string, int>;
using FuncPtr = void (*)(int, double);                // 比 typedef 版直观太多了！

// ===== using 对模板别名也支持（typedef 做不到！） =====
template <typename T>
using Tensor1D = vector<T>;                // 一维张量就是 vector

template <typename T>
using Tensor2D = vector<vector<T>>;        // 二维张量

// template <typename T>
// typedef vector<T> Tensor1D_Typedef;      // ❌ typedef 不能用于模板别名，编译报错！

int main() {
    // ===== 使用类型别名 =====
    Tensor1D<double> weights(100);           // 等价于 vector<double> weights(100);
    Tensor2D<float> featureMap(28, vector<float>(28));  // 28×28 的特征图

    StringIntMap scores;
    scores["张三"] = 88;
    scores["李四"] = 93;

    cout << "weights 大小：" << weights.size() << endl;
    cout << "featureMap 大小：" << featureMap.size() << "×"
         << featureMap[0].size() << endl;
    cout << "张三成绩：" << scores["张三"] << endl;

    return 0;
}
```

#### 用途二：命名空间引入

```cpp
// ① 引入整个命名空间（我们在所有示例代码里都在用）
using namespace std;    // 引入 std 命名空间下的所有名字

// ② 引入单个名字（比引入整个命名空间更精确，推荐在头文件中使用）
using std::cout;
using std::endl;
using std::vector;
using std::string;

// ③ 子命名空间别名
namespace fs = std::filesystem;             // C++17 文件系统
namespace chr = std::chrono;                // 时间库
// 之后写 fs::path、chr::seconds 即可，不用写一长串 std::filesystem::path
```

> 💡 **`typedef` vs `using`**：能用 `using` 的地方就别用 `typedef`。`using` 的语法读起来更自然——`using 别名 = 原类型;`，而且支持模板别名，这是 `typedef` 做不到的。在 InfiniTensor 这类模板密集的框架中，`using` 几乎是刚需。

---

## 📝 第十一部分：全文总结

这篇文章从 Vector 扩容出发，一路打通了 C++ 高阶面向对象、内存规范、STL 体系与现代语法。来梳理全部核心收获：

| 板块 | 核心收获 |
|------|---------|
| **Vector 自动扩容** | 理解了 `size` vs `capacity` 的本质区别，掌握了「分配 → 拷贝 → 释放 → 更新指针」四步扩容流程，知道了 2 倍扩容策略背后的均摊 O(1) 时间复杂度原理 |
| **双重释放问题** | 搞清楚了浅拷贝导致双重释放的完整链条——默认拷贝构造让两个指针指向同一块内存，析构时两次 `delete` 直接崩溃 |
| **初始化列表** | 掌握了初始化（一步到位）vs 赋值（先默认构造再覆盖）的本质区别，知道 `const` 成员、引用成员**必须**用初始化列表 |
| **三法则 Rule of Three** | 「有堆指针，三法全写」——析构释放、拷贝构造独立分配、拷贝赋值先释放再分配。缺失任何一个都会导致泄漏或崩溃 |
| **五法则 Rule of Five** | 在三大法则基础上增加移动构造和移动赋值——「偷」走即将销毁对象的资源，避免无谓的分配和拷贝，大张量场景下性能提升可达几十倍 |
| **STL 标准容器** | 系统对比了 `vector`、`deque`、`list`、`map`、`unordered_map` 的底层结构、时间复杂度和适用场景，掌握了 AI 开发中的容器选型三问 |
| **继承与多态** | 理解了 `public` 继承的访问权限规则、虚函数表（vtable）的多态实现原理、纯虚函数与抽象类的接口定义能力——这是大模型框架插件化架构的基础 |
| **智能指针** | `unique_ptr`（独占、自动释放）和 `shared_ptr`（共享、引用计数）的核心用法——现代 C++ 的「告别手动 delete」之道 |
| **auto 与 using** | `auto` 自动类型推导让迭代器和模板代码清爽十倍，`using` 类型别名比 `typedef` 更直观且支持模板别名 |

这些知识不是孤立的——它们是编写**高性能、安全、工程级 C++ 张量数据结构**的必备能力。回顾一下完整链条：

- **扩容**让你的容器能应对不可预知的数据规模——推理请求 batch size 可变、序列长度可变，容器必须跟着变
- **三/五法则**让你的类在拷贝、赋值、析构、移动时行为正确——张量对象在函数之间传递、被容器管理，每一步都不能泄漏或崩溃
- **STL 容器**让你站在工业级实现的基础上工作——不用重复造轮子，集中精力解决 AI 领域问题
- **继承与多态**让你设计出可扩展的算子体系——框架能不断接入新算子而不改核心代码
- **智能指针**让你告别手动内存管理——RAII 自动保证资源释放，专注算法逻辑
- **现代关键字**让你的代码更简洁、更安全、更 C++11+——跟上语言进化的步伐

这些能力将在后续 InfiniTensor 大模型系统开发中直接支撑：**张量运算的实现、内存池的管理、算子注册与调度、模型权重的加载与共享**。

---

## 📖 系列导航

从 WSL 环境搭建 → C++ 基础语法 → Git 与进阶语法 → 面向对象与模板 → **Vector 扩容与 C++ 高阶特性**，我们已经完成了训练营 C++ 模块的五站旅程。每一篇都环环相扣、层层递进，下一站我们将继续深入更高级的 C++ 工程实践与 InfiniTensor 框架实战。

> 🚀 五大法则、智能指针、多态抽象——这些是区分「能写 C++」和「写好 C++」的分水岭。理解它们，带着它们进入大模型框架的世界，你写出的每一行张量代码都会更安全、更高效、更专业！

📖 全系列文章入口：
- 个人博客：[https://liuyuqi-qiqiqi.github.io](https://liuyuqi-qiqiqi.github.io)
- 源码与博客仓库：[https://github.com/liuyuqi-qiqiqi/liuyuqi-qiqiqi.github.io](https://github.com/liuyuqi-qiqiqi/liuyuqi-qiqiqi.github.io)

更多2026夏季InfiniTensor训练营系列学习笔记持续更新，欢迎访问我的个人博客：https://liuyuqi-qiqiqi.github.io
所有文章源码与文档托管于GitHub仓库：https://github.com/liuyuqi-qiqiqi/liuyuqi-qiqiqi.github.io

---

**个人博客地址**：https://liuyuqi-qiqiqi.github.io
**个人GitHub仓库地址**：https://github.com/liuyuqi-qiqiqi/liuyuqi-qiqiqi.github.io
