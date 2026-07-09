---
title: 2026年夏季InfiniTensor训练营｜动态数组Vector实现、C++面向对象与模板类进阶
date: 2026-07-10 15:00:00
tags: 2026训练营,InfiniTensor,C++,动态数组,内存管理,类与对象,模板类
categories: 2026夏季InfiniTensor训练营
cover:
description: 本篇从零手写简易Vector动态数组，掌握堆内存分配与释放、Valgrind内存检测、类与访问控制、构造析构函数、运算符重载、文件分离编译、C++模板类核心知识。
---

# 🧱 2026年夏季InfiniTensor训练营｜动态数组Vector实现、C++面向对象与模板类进阶

[上一篇文章](https://liuyuqi-qiqiqi.github.io/2026/07/08/2026-InfiniTensor-training-camp-Git-Cpp-advanced/) 我们搞定了 Git 版本管理、clang-format 代码格式化，还深入学习了 const 常量、静态数组、指针和结构体这些 C++ 进阶语法。这些是基本功——**但真正的工程开发，代码量和复杂度远超一个 `main.cpp` 能装下的范畴**。

在 InfiniTensor 这类大模型推理框架中，你需要管理成千上万个张量的内存分配与释放，你需要用类来组织复杂的模块，你需要写出能适配 `float`、`double`、`int` 等各种类型的通用数据结构。今天这篇文章，我们就来打通这几条 C++ 工程核心命脉：**动态内存管理**、**面向对象编程**、**模板编程**——并通过**从零手写一个简易 Vector 动态数组**，把所有知识点串联起来。

内容很硬核，但每一节都配了可运行的代码。打开你的 WSL 终端，我们开始！

---

## 💾 第一部分：动态内存管理——告别「数组大小写死」的痛苦

### 1.1 静态数组的先天缺陷

[上一篇](https://liuyuqi-qiqiqi.github.io/2026/07/08/2026-InfiniTensor-training-camp-Git-Cpp-advanced/) 3.2 节我们学习了静态数组，它能批量管理同类型数据，但有一个致命短板——**大小必须在编译时确定**：

```cpp
// ❌ 静态数组的尴尬：大小不能是变量
int n;
cin >> n;
// int arr[n];   // 某些编译器可能不报错，但这不是标准 C++，行为不可靠！

// ✅ 只能这样：先预估一个最大值
int arr[1000];   // 如果用户只需要 10 个，浪费 990 个空间
                 // 如果用户需要 2000 个，不好意思——越界了，程序崩溃
```

在实际工程中，数据量几乎不可能在编译时预知。你不可能写一个推理框架，然后说「最大支持 1000 个张量，多了不伺候」——**用户的数据规模是运行时的变量，你的数据结构也要能跟随变化**。

### 1.2 栈内存 vs 堆内存

要理解动态数组，首先要分清两种内存区域：

| 特性 | **栈内存（Stack）** | **堆内存（Heap）** |
|------|-------------------|-------------------|
| 分配方式 | 编译器自动分配、自动释放 | 程序员手动 `new` / `malloc` 申请 |
| 大小限制 | 通常较小（几 MB），超出会栈溢出 | 受系统总内存限制，可分配 GB 级别 |
| 生命周期 | 离开作用域 `{}` 自动销毁 | 程序员决定何时释放（`delete` / `free`） |
| 速度 | 极快（移动栈指针即可） | 相对较慢（需要在空闲链表中查找） |
| 典型用途 | 局部变量、函数参数、小数组 | 动态数组、大对象、生命周期跨函数的数据 |

用生活类比来理解：**栈**是写字台，空间有限但随用随拿，起身离开就自动收好；**堆**是仓库，空间大但需要你亲自登记入库、亲自清退出库——忘了清退就是内存泄漏。

### 1.3 new / delete：C++ 动态内存的核心操作

在 C++ 中，**`new`** 在堆上分配内存，**`delete`** 释放那块内存。它们必须成对出现：

```cpp
#include <iostream>
using namespace std;

int main() {
    // ===== 单个变量的动态分配 =====
    int *p = new int;      // 在堆上分配一个 int，p 指向它
    *p = 42;
    cout << "*p = " << *p << endl;   // 输出：42
    delete p;              // 用完了，释放内存
    // 此时 p 仍然存着那个地址，但内存已经不属于你了
    p = nullptr;           // ✅ 好习惯：释放后立即置空，防止悬垂指针

    // ===== 动态数组：new[] 和 delete[] =====
    int size;
    cout << "请输入数组大小: ";
    cin >> size;

    int *arr = new int[size];   // 在堆上分配 size 个 int 的连续空间

    // 初始化并输出
    for (int i = 0; i < size; i++) {
        arr[i] = i * 10;        // 和普通数组完全一样的访问方式
    }
    for (int i = 0; i < size; i++) {
        cout << arr[i] << " ";  // 输出：0 10 20 30 ...（取决于 size）
    }
    cout << endl;

    delete[] arr;    // ⚠️ 动态数组必须用 delete[]，不能用 delete！
    arr = nullptr;   // 释放后置空

    return 0;
}
```

> ⚠️ **配对铁律**：`new` 配 `delete`，`new[]` 配 `delete[]`。混用会导致未定义行为——编译器不会报错，但程序可能随机崩溃或静默泄漏内存，这是最让人头疼的 bug。

### 1.4 内存泄漏：沉默的性能杀手

**内存泄漏（Memory Leak）**是指你用 `new` 申请了堆内存，但用完之后忘了 `delete`。那块内存既没有被释放，也失去了指向它的指针——**永远无法回收**。

```cpp
// ❌ 典型的内存泄漏场景
void leakyFunction() {
    int *data = new int[1000000];   // 申请了 4MB 堆内存（假设 int 是 4 字节）
    // 处理数据……
    // 函数结束，局部变量 data 被销毁
    // 但是！data 指向的 1000000 个 int 还留在堆上，永远收不回来了
    // 忘记 delete[] data;
}

int main() {
    for (int i = 0; i < 100; i++) {
        leakyFunction();   // 每次调用泄漏 4MB，100 次就是 400MB
    }
    // 程序占用了 400MB 额外内存，操作系统只能等程序退出时统一回收
    return 0;
}
```

在大模型推理中，一个推理请求可能涉及数百 MB 甚至 GB 级的张量数据。如果一个推理循环中泄漏了内存没释放，**几分钟之内内存就会被撑爆，服务直接 OOM（Out of Memory）崩溃**。这也是为什么我们要用专门的工具来检测内存问题——下一节马上讲。

---

## 🔍 第二部分：Valgrind 内存调试工具——给代码做「内存体检」

### 2.1 Valgrind 是什么？

**Valgrind** 是 Linux 下最权威的内存调试工具之一。它能在不修改源代码的情况下，监控程序的每一次内存分配、访问和释放，帮你揪出三种常见的内存问题：

| 检测能力 | 说明 | 危害程度 |
|---------|------|---------|
| **内存泄漏** | `new` 了但没 `delete`，堆内存永远丢失 | ⭐⭐⭐⭐⭐ 长期运行必出事 |
| **野指针 / 非法访问** | 访问了已释放的内存，或访问了未分配的地址 | ⭐⭐⭐⭐⭐ 概率性崩溃 |
| **重复释放** | 对同一块内存 `delete` 了两次 | ⭐⭐⭐⭐ 直接 crash |

### 2.2 WSL Ubuntu 下安装 Valgrind

```bash
# 安装 Valgrind
sudo apt install -y valgrind

# 确认安装成功
valgrind --version
# 输出示例：valgrind-3.22.0
```

### 2.3 基本使用：检测一个程序的内存情况

Valgrind 的使用方式非常简洁——在你要运行的命令前加上 `valgrind` 即可：

```bash
# 基本格式
valgrind --leak-check=full ./你的程序
```

先来看一个「干净」的程序，熟悉正常输出长什么样：

```cpp
// clean.cpp —— 没有内存问题
#include <iostream>
using namespace std;

int main() {
    int *p = new int(100);
    cout << "*p = " << *p << endl;
    delete p;         // ✅ 正确释放
    p = nullptr;
    return 0;
}
```

```bash
# 编译（加上 -g 以便 Valgrind 定位源码行号）
g++ -g clean.cpp -o clean

# Valgrind 检测
valgrind --leak-check=full ./clean
```

正常输出中，最关键的几行：

```
==123== HEAP SUMMARY:
==123==     in use at exit: 0 bytes in 0 blocks      ← 退出时堆上没残留 ✅
==123==   total heap usage: 1 allocs, 1 frees, 4 bytes allocated
==123==
==123== All heap blocks were freed -- no leaks are possible   ← 满分！
```

> 💡 **解读**：`0 bytes in 0 blocks` + `no leaks are possible` = 内存管理满分。每次 `new` 都对应了 `delete`，没有任何泄漏。

### 2.4 实战：用 Valgrind 抓出内存泄漏

下面是一段故意写了三个内存问题的「坏代码」，我们用 Valgrind 来逐一揪出它们：

```cpp
// leaky.cpp —— 故意埋了三个内存问题
#include <iostream>
using namespace std;

int main() {
    // 问题 1：内存泄漏 —— 分配了内存但没释放
    int *leak = new int[100];
    for (int i = 0; i < 100; i++) {
        leak[i] = i;
    }
    // 忘记 delete[] leak;  ❌

    // 问题 2：野指针 —— 访问已释放的内存
    int *p = new int(42);
    delete p;
    // cout << *p << endl;   // ❌ p 已经是悬垂指针！取消注释会出问题

    // 问题 3：new 和 new[] 混用
    int *arr = new int[10];
    // delete arr;           // ❌ 应该用 delete[] arr
    delete[] arr;            // ✅ 正确配对

    return 0;
}
```

```bash
# 编译（务必加 -g 调试符号）
g++ -g leaky.cpp -o leaky

# Valgrind 检测
valgrind --leak-check=full ./leaky
```

Valgrind 会输出类似这样的泄漏报告：

```
==1234== HEAP SUMMARY:
==1234==     in use at exit: 400 bytes in 1 blocks
==1234==   total heap usage: 3 allocs, 2 frees, 416 bytes allocated
==1234==
==1234== 400 bytes in 1 blocks are definitely lost in loss record 1 of 1
==1234==    at 0x484A2F3: operator new[](unsigned long) (in /usr/lib/...)
==1234==    by 0x1091FE: main (leaky.cpp:7)     ← 精确指出 leaky.cpp 第 7 行！
==1234==
==1234== LEAK SUMMARY:
==1234==    definitely lost: 400 bytes in 1 blocks   ← 确定泄漏了 400 字节
```

> 🎯 Valgrind 不仅告诉你「有泄漏」，还精确到**文件名和行号**——这就是 `-g` 编译选项的功劳。它直接帮你定位到 `leaky.cpp:7`，也就是 `leak = new int[100]` 那一行。

### 2.5 Valgrind 常用选项速查

```bash
# 全面检查（推荐日常使用）
valgrind --leak-check=full --show-leak-kinds=all ./程序名

# 追踪未初始化的值
valgrind --track-origins=yes ./程序名

# 生成详细报告并保存到文件
valgrind --leak-check=full --log-file=valgrind_report.txt ./程序名

# 只看泄漏摘要（更简洁）
valgrind --leak-check=summary ./程序名
```

| 选项 | 作用 |
|------|------|
| `--leak-check=full` | 完整泄漏检测，输出每个泄漏点的详细信息 |
| `--show-leak-kinds=all` | 报告所有类型的泄漏（包括间接泄漏和可能泄漏） |
| `--track-origins=yes` | 追踪未初始化变量的来源（排查「使用了未初始化值」的 bug） |
| `--log-file=文件名` | 把报告写入文件，不混在程序输出里 |
| `-g`（编译选项）| 在可执行文件中加入调试信息，Valgrind 才能给出行号 |

> 💡 养成习惯：写完任何涉及动态内存的代码，先用 `valgrind --leak-check=full` 跑一遍，确认 `All heap blocks were freed` 再提交。在大模型工程中，**宁可多花两分钟做内存检查，也不要在生产环境半夜被 OOM 报警叫起来**。

---

## 🏛️ 第三部分：C++ 类与面向对象——把数据和操作打包在一起

[上一篇](https://liuyuqi-qiqiqi.github.io/2026/07/08/2026-InfiniTensor-training-camp-Git-Cpp-advanced/) 我们学了结构体 `struct`，能把姓名、学号、成绩这些相关字段打包成一个类型。**类（class）**是结构体的「升级版」——它不仅打包数据，还打包操作这些数据的函数，实现**封装**。

### 3.1 从 struct 到 class：一扇新世界的大门

先看一个对比，直观感受 class 和 struct 的区别：

```cpp
// struct 版本：数据是裸露的，谁都能直接改
struct StudentStruct {
    string name;
    int id;
    double score;
};
// 外部可以随意：stu.score = -999;  // ❌ 不合法的成绩，struct 管不了

// class 版本：数据私有，通过公开的成员函数来安全地操作
class Student {
private:
    string name;       // 外部不能直接访问
    int id;
    double score;

public:
    // 通过公开函数来设置成绩——加入合法性检查
    void setScore(double s) {
        if (s >= 0 && s <= 100) {
            score = s;
        } else {
            cout << "⚠️ 成绩必须在 0~100 之间！" << endl;
        }
    }

    double getScore() const {   // const 表示这个函数不会修改成员变量
        return score;
    }
};
```

> 🎯 **核心思想**：`private` 成员是「内部实现细节」，外部代码碰不到；`public` 成员函数是「对外接口」，外部通过它来安全地交互。这就像 ATM 机——你只能通过屏幕按钮（public 接口）操作，不能直接伸手进机器里掏钱（private 数据）。

### 3.2 类的基本语法

```cpp
#include <iostream>
#include <string>
using namespace std;

class Person {
// ===== 私有成员：只能在类内部访问 =====
private:
    string name;
    int age;

// ===== 公有成员：外部可以访问 =====
public:
    // 设置姓名
    void setName(const string &n) {
        name = n;
    }

    // 获取姓名（const 承诺不修改任何成员变量）
    string getName() const {
        return name;
    }

    // 设置年龄（带合法性校验）
    void setAge(int a) {
        if (a >= 0 && a <= 150) {
            age = a;
        } else {
            cout << "⚠️ 年龄不合法！" << endl;
        }
    }

    // 打印信息
    void printInfo() const {
        cout << "姓名: " << name << ", 年龄: " << age << endl;
    }
};

int main() {
    Person p1;
    p1.setName("张三");
    p1.setAge(25);
    p1.printInfo();          // 输出：姓名: 张三, 年龄: 25

    // p1.name = "李四";      // ❌ 编译报错！name 是 private
    // p1.age = -5;           // ❌ 编译报错！就算不报错，setAge 也会拦截 -5

    p1.setAge(-5);            // ✅ 编译通过，但运行时会输出「年龄不合法」
    p1.printInfo();           // 年龄仍然是 25，不会被改成 -5

    return 0;
}
```

### 3.3 构造函数：对象诞生的那一刻

每次创建对象，C++ 会自动调用一个特殊函数——**构造函数（Constructor）**。它负责初始化对象，函数名和类名相同，**没有返回值**（连 `void` 都不写）。

```cpp
#include <iostream>
#include <string>
using namespace std;

class Student {
private:
    string name;
    int id;
    double score;

public:
    // ===== 构造函数 1：默认构造函数（无参数） =====
    Student() {
        name = "未命名";
        id = 0;
        score = 0.0;
        cout << "默认构造函数被调用 —— 一个学生对象诞生了！" << endl;
    }

    // ===== 构造函数 2：带参构造函数 =====
    Student(string n, int i, double s) {
        name = n;
        id = i;
        score = s;
        cout << "带参构造函数被调用 —— " << name << " 已注册！" << endl;
    }

    void printInfo() const {
        cout << "学号: " << id << "  姓名: " << name << "  成绩: " << score << endl;
    }
};

int main() {
    Student s1;                              // 调用默认构造函数
    s1.printInfo();                          // 输出默认值

    Student s2("张三", 2026001, 88.5);       // 调用带参构造函数
    s2.printInfo();

    // 也可以用 new 在堆上创建（调用带参构造）
    Student *s3 = new Student("李四", 2026002, 93.0);
    s3->printInfo();
    delete s3;

    return 0;
}
```

运行输出：

```
默认构造函数被调用 —— 一个学生对象诞生了！
学号: 0  姓名: 未命名  成绩: 0
带参构造函数被调用 —— 张三 已注册！
学号: 2026001  姓名: 张三  成绩: 88.5
带参构造函数被调用 —— 李四 已注册！
学号: 2026002  姓名: 李四  成绩: 93
```

> 💡 **成员初始化列表**（更高效的方式）：上面的带参构造函数用了「先赋默认值再在函数体里覆盖」的方式。对于基础类型没问题，但对于 `string` 这类对象会多一次不必要的默认构造。推荐用**初始化列表**：
>
> ```cpp
> // ✅ 初始化列表：冒号后面直接初始化，一步到位
> Student(string n, int i, double s) : name(n), id(i), score(s) {
>     cout << "带参构造函数被调用 —— " << name << " 已注册！" << endl;
> }
> ```

### 3.4 析构函数：对象销毁前的最后一口气

**析构函数（Destructor）**在对象生命周期结束时自动调用，函数名是 `~类名`，同样**没有返回值、没有参数**。它最常见的用途是**释放对象持有的堆内存**——确保不泄漏。

```cpp
#include <iostream>
#include <string>
using namespace std;

class DynamicArray {
private:
    int *data;      // 指向堆上动态分配的数组
    int size;

public:
    // 构造函数：在堆上分配数组
    DynamicArray(int s) : size(s) {
        data = new int[size];
        cout << "构造函数：在堆上分配了 " << size << " 个 int（"
             << (size * sizeof(int)) << " 字节）" << endl;
    }

    // 析构函数：释放堆内存
    ~DynamicArray() {
        delete[] data;
        data = nullptr;
        cout << "析构函数：释放了 " << size << " 个 int 的堆内存" << endl;
    }

    void setValue(int index, int value) {
        if (index >= 0 && index < size) {
            data[index] = value;
        }
    }

    int getValue(int index) const {
        return (index >= 0 && index < size) ? data[index] : -1;
    }
};

int main() {
    cout << "=== 进入 main 函数 ===" << endl;

    {
        cout << "进入内层作用域..." << endl;
        DynamicArray arr(5);             // 构造函数在这调用
        arr.setValue(0, 100);
        cout << "arr[0] = " << arr.getValue(0) << endl;
        cout << "即将离开内层作用域..." << endl;
    }  // ← arr 在这里离开作用域，析构函数自动调用，堆内存自动释放

    cout << "=== 已离开内层作用域 ===" << endl;
    return 0;
}
```

运行输出：

```
=== 进入 main 函数 ===
进入内层作用域...
构造函数：在堆上分配了 5 个 int（20 字节）
arr[0] = 100
即将离开内层作用域...
析构函数：释放了 5 个 int 的堆内存
=== 已离开内层作用域 ===
```

> 🎯 **关键理解**：你看到了吗？程序里**没有任何显式的 `delete[]` 调用**——因为析构函数帮你做了这件事。这就是 C++ RAII（Resource Acquisition Is Initialization，资源获取即初始化）思想的核心：**在构造函数中获取资源，在析构函数中自动释放资源**。对象离开作用域的那一刻，资源就自动还回去了，绝不会忘。

| 构造 / 析构 | 何时调用 | 主要职责 |
|------------|---------|---------|
| **构造函数** `ClassName()` | 对象创建时 | 初始化成员变量、申请堆内存等资源 |
| **析构函数** `~ClassName()` | 对象销毁时（离开作用域 / `delete`） | 释放堆内存、关闭文件、断开连接 |

---

## 🔣 第四部分：运算符重载——让你的类像内置类型一样自然

### 4.1 什么是运算符重载？

C++ 内置类型可以直接用 `+`、`-`、`=`、`[]` 等运算符。但对于我们自己定义的类，编译器不知道这些运算符该怎么处理。**运算符重载**就是让你告诉编译器：「如果有人说 `a + b`，其中 `a` 和 `b` 都是我的类，请这样做……」

```cpp
// 没有运算符重载的话：代码很啰嗦
vec1.add(vec2);        // 向量相加需要调用函数
int x = vec.get(3);    // 访问元素也要调用函数

// 有了运算符重载：代码自然、直观
Vector v3 = vec1 + vec2;   // 和 built-in 类型一样的写法
int x = vec[3];             // 用 [] 直接访问，就像普通数组
```

对于我们要实现的 Vector 动态数组来说，最重要的两个重载是**赋值运算符 `=`** 和**下标运算符 `[]`**。

### 4.2 下标运算符 `[]`：像原生数组一样访问

```cpp
#include <iostream>
using namespace std;

class IntArray {
private:
    int *data;
    int size;

public:
    IntArray(int s) : size(s) {
        data = new int[size];
        for (int i = 0; i < size; i++) {
            data[i] = 0;    // 初始化为 0
        }
    }

    ~IntArray() {
        delete[] data;
    }

    // ===== 下标运算符重载：可读可写版本 =====
    int &operator[](int index) {
        return data[index];   // 返回引用，外部可以直接修改
    }

    // ===== 下标运算符重载：只读版本（给 const 对象用的） =====
    const int &operator[](int index) const {
        return data[index];
    }

    int getSize() const { return size; }
};

int main() {
    IntArray arr(5);

    // 用 [] 赋值 —— 就像原生数组一样！
    for (int i = 0; i < arr.getSize(); i++) {
        arr[i] = i * 10;     // 等价于 arr.operator[](i) = i * 10
    }

    // 用 [] 读取
    for (int i = 0; i < arr.getSize(); i++) {
        cout << "arr[" << i << "] = " << arr[i] << endl;
    }
    // 输出：
    // arr[0] = 0
    // arr[1] = 10
    // arr[2] = 20
    // arr[3] = 30
    // arr[4] = 40

    return 0;
}
```

> 💡 `operator[]` 返回 `int &`（引用）是关键——返回引用意味着 `arr[0] = 100` 时，赋值的是 `data[0]` 这块真实内存，而不是一个临时的副本。如果返回的不是引用，赋值就无效了。

### 4.3 赋值运算符 `=`：正确处理深拷贝

```cpp
#include <iostream>
using namespace std;

class IntArray {
private:
    int *data;
    int size;

public:
    IntArray(int s) : size(s) {
        data = new int[size]();    // () 初始化为全 0
    }

    // 拷贝构造函数（创建新对象时用已有对象初始化）
    IntArray(const IntArray &other) : size(other.size) {
        data = new int[size];
        for (int i = 0; i < size; i++) {
            data[i] = other.data[i];
        }
        cout << "拷贝构造函数被调用" << endl;
    }

    // ===== 赋值运算符重载 =====
    IntArray &operator=(const IntArray &other) {
        cout << "赋值运算符被调用" << endl;

        // 步骤 1：防止自我赋值（a = a 是无意义操作）
        if (this == &other) {
            return *this;
        }

        // 步骤 2：释放自己的旧资源
        delete[] data;

        // 步骤 3：根据右侧对象的大小重新分配
        size = other.size;
        data = new int[size];

        // 步骤 4：逐元素拷贝
        for (int i = 0; i < size; i++) {
            data[i] = other.data[i];
        }

        return *this;    // 返回自身引用，支持 a = b = c 的链式赋值
    }

    ~IntArray() {
        delete[] data;
    }

    int &operator[](int index) { return data[index]; }
    int getSize() const { return size; }
};

int main() {
    IntArray a(3);
    a[0] = 10; a[1] = 20; a[2] = 30;

    IntArray b(5);      // b 的大小是 5
    b = a;              // 赋值后，b 的大小变成了 3，内容是 a 的拷贝

    // 修改 a 不会影响 b —— 因为赋值运算符做了真正的「深拷贝」
    a[0] = 999;
    cout << "a[0] = " << a[0] << endl;   // 999
    cout << "b[0] = " << b[0] << endl;   // 10（没有被影响！）

    return 0;
}
```

> ⚠️ **浅拷贝 vs 深拷贝**：如果不用自定义的 `operator=`，C++ 会生成默认的赋值运算符——它只做「逐成员复制」（浅拷贝），这意味着 `b.data = a.data`（两个指针指向同一块内存）。结果呢？修改 `a[0]` 会影响 `b[0]`，析构时同一块内存被 `delete[]` 两次，程序直接崩溃。**凡是类里有指向堆内存的指针，必须自己写赋值运算符和拷贝构造函数！**

---

## 📁 第五部分：头文件与源文件分离编译——告别「单文件怪兽」

### 5.1 为什么要把代码分到多个文件里？

前面的示例代码全部挤在一个 `.cpp` 文件里。对于几十行的小程序无所谓，但在真实工程中——InfiniTensor 有几十万行代码——**全部写在一个文件里是不可想象的**。代码分离带来的好处非常直接：

| 好处 | 说明 |
|------|------|
| **可维护性** | 想改 Vector 的实现，只需打开 `Vector.cpp`，不用在几万行的文件里来回翻 |
| **并行开发** | 张三写 Vector，李四写 Matrix，互不冲突 |
| **编译效率** | 只改了 `Vector.cpp`，只需重新编译这一个文件再链接，不用全量重编 |
| **接口与实现分离** | 使用者看 `.h` 头文件就知道怎么用，不需要关心 `.cpp` 里怎么实现的 |

### 5.2 分离的基本规则

C++ 工程的约定俗称是这样的：

| 文件类型 | 放什么 | 示例 |
|---------|--------|------|
| **`.h` 头文件** | 类声明、函数声明、宏定义、类型定义（**接口**） | `Vector.h` |
| **`.cpp` 源文件** | 类成员函数的实现、普通函数的实现（**具体代码**） | `Vector.cpp` |
| **`main.cpp`** | `main()` 函数，程序的入口 | `main.cpp` |

### 5.3 实战：把一个类拆成 .h 和 .cpp

**Vector.h** —— 头文件，只放声明：

```cpp
#ifndef VECTOR_H        // 防止重复包含的「头文件守卫」
#define VECTOR_H

class Vector {
private:
    int *data;          // 指向堆上动态数组的指针
    int size;           // 当前元素数量
    int capacity;       // 当前分配的容量（可容纳的最大元素数）

public:
    // 构造函数
    Vector(int cap = 4);    // 默认容量为 4

    // 析构函数
    ~Vector();

    // 在末尾添加元素
    void pushBack(int value);

    // 下标访问
    int &operator[](int index);

    // 获取大小
    int getSize() const;

    // 获取容量
    int getCapacity() const;

private:
    // 扩容函数（内部使用，不对外暴露）
    void resize();
};

#endif   // VECTOR_H
```

**Vector.cpp** —— 源文件，放具体实现：

```cpp
#include "Vector.h"
#include <iostream>
using namespace std;

// 构造函数：分配初始内存
Vector::Vector(int cap) : size(0), capacity(cap) {
    data = new int[capacity];
    cout << "Vector 构造：容量 = " << capacity << endl;
}

// 析构函数：释放堆内存
Vector::~Vector() {
    delete[] data;
    data = nullptr;
    cout << "Vector 析构：内存已释放" << endl;
}

// 添加元素
void Vector::pushBack(int value) {
    // 满了就扩容
    if (size >= capacity) {
        resize();
    }
    data[size] = value;
    size++;
}

// 下标访问
int &Vector::operator[](int index) {
    return data[index];
}

// 获取大小
int Vector::getSize() const {
    return size;
}

// 获取容量
int Vector::getCapacity() const {
    return capacity;
}

// 扩容：容量翻倍
void Vector::resize() {
    int newCapacity = capacity * 2;
    int *newData = new int[newCapacity];   // 在堆上分配更大的新数组

    // 把旧数据拷贝到新数组
    for (int i = 0; i < size; i++) {
        newData[i] = data[i];
    }

    delete[] data;        // 释放旧数组
    data = newData;       // 指针指向新数组
    capacity = newCapacity;

    cout << "⚡ Vector 扩容：" << (capacity / 2)
         << " → " << capacity << endl;
}
```

**main.cpp** —— 入口文件：

```cpp
#include "Vector.h"
#include <iostream>
using namespace std;

int main() {
    Vector vec(2);    // 初始容量 2，方便观察扩容

    // 添加 10 个元素，观察自动扩容
    for (int i = 0; i < 10; i++) {
        vec.pushBack(i * 10);
    }

    // 遍历输出
    cout << "\n=== Vector 内容 ===" << endl;
    for (int i = 0; i < vec.getSize(); i++) {
        cout << "vec[" << i << "] = " << vec[i] << endl;
    }

    cout << "\n最终大小: " << vec.getSize()
         << "，最终容量: " << vec.getCapacity() << endl;

    return 0;
}
```

### 5.4 多文件编译

```bash
# 方式 1：逐个编译再链接
g++ -c Vector.cpp -o Vector.o      # -c 只编译，不链接
g++ -c main.cpp -o main.o
g++ Vector.o main.o -o program     # 链接所有 .o 为目标可执行文件

# 方式 2：一条命令搞定（推荐，适合文件少的时候）
g++ Vector.cpp main.cpp -o program

# 运行
./program
```

运行效果：

```
Vector 构造：容量 = 2
⚡ Vector 扩容：2 → 4
⚡ Vector 扩容：4 → 8
⚡ Vector 扩容：8 → 16

=== Vector 内容 ===
vec[0] = 0
vec[1] = 10
vec[2] = 20
vec[3] = 30
vec[4] = 40
vec[5] = 50
vec[6] = 60
vec[7] = 70
vec[8] = 80
vec[9] = 90

最终大小: 10，最终容量: 16
Vector 析构：内存已释放
```

### 5.5 头文件守卫（Header Guard）详解

`Vector.h` 里的三行：

```cpp
#ifndef VECTOR_H
#define VECTOR_H
// ... 头文件内容 ...
#endif
```

这三行叫做**头文件守卫**，用于防止同一个头文件被多次包含。场景是这样的：

```cpp
// A.h 包含了 Vector.h
// B.h 也包含了 Vector.h
// main.cpp 同时包含了 A.h 和 B.h
// → Vector.h 被包含了两次 → 编译器报「重复定义」错误

// 有 #ifndef 守卫后：
// 第一次包含 → VECTOR_H 未定义 → 定义它 → 展开内容
// 第二次包含 → VECTOR_H 已定义 → 直接跳过全部内容
```

> 🎯 规范：**每个 `.h` 头文件都必须加守卫**，宏名习惯用 `文件名全大写_H`。比如 `Vector.h` → `VECTOR_H`，`Tensor.h` → `TENSOR_H`。

---

## 🧩 第六部分：C++ 模板——写出「一次定义，处处适用」的通用代码

### 6.1 模板解决什么问题？

看一个尴尬的场景——如果我们想让 Vector 支持 `double` 类型：

```cpp
// ❌ 笨办法：为每种类型各写一个类
class IntVector {    // int 版本 —— 200 行
    int *data;
    // ... 所有逻辑 ...
};

class DoubleVector { // double 版本 —— 内容几乎一模一样，但又要写 200 行
    double *data;
    // ... 所有逻辑完全重复 ...
};

// 如果还要支持 float、char、string、自定义类型呢？
// 无穷无尽的复制粘贴 → 代码膨胀、维护灾难
```

**模板（Template）**就是用来消灭这种重复的。你只需要写**一份**代码，编译器会根据实际使用的类型，自动帮你「套用」出对应版本。

### 6.2 函数模板入门

```cpp
#include <iostream>
using namespace std;

// ===== 函数模板：一个函数，支持任意类型 =====
template <typename T>    // T 是「类型参数」，你可以叫它任何名字（常见 T、U、Type）
T maxValue(T a, T b) {
    return (a > b) ? a : b;
}

int main() {
    // 编译器自动推导 T 的类型
    cout << maxValue(10, 20) << endl;            // T = int     → 输出 20
    cout << maxValue(3.14, 2.71) << endl;        // T = double  → 输出 3.14
    cout << maxValue('x', 'a') << endl;          // T = char    → 输出 x

    // 也可以显式指定类型
    cout << maxValue<double>(5, 3.9) << endl;    // T = double  → 输出 5.0

    return 0;
}
```

> 💡 **模板的工作原理**：模板本身不是代码，它是「代码的模具」。编译器看到 `maxValue(10, 20)` 时，发现 `10` 和 `20` 都是 `int`，于是用模具「冲压」出一份 `T = int` 的版本。这个冲压过程发生在**编译期**，不会影响运行效率。

### 6.3 类模板：让我们的 Vector 支持任意数据类型

把上一节的 Vector 改成模板类，改动比你想象的要小得多：

```cpp
// Vector.h —— 模板类版本
#ifndef VECTOR_H
#define VECTOR_H

#include <iostream>
using namespace std;

template <typename T>         // ← 类模板声明
class Vector {
private:
    T *data;                  // ← T 类型的指针
    int size;
    int capacity;

    void resize() {
        int newCapacity = capacity * 2;
        T *newData = new T[newCapacity];   // ← new T[]，分配 T 类型的数组

        for (int i = 0; i < size; i++) {
            newData[i] = data[i];
        }

        delete[] data;
        data = newData;
        capacity = newCapacity;

        cout << "⚡ Vector<" << typeid(T).name()
             << "> 扩容：" << (capacity / 2)
             << " → " << capacity << endl;
    }

public:
    Vector(int cap = 4) : size(0), capacity(cap) {
        data = new T[capacity];
    }

    ~Vector() {
        delete[] data;
    }

    void pushBack(const T &value) {     // const 引用，高效传递任意类型
        if (size >= capacity) {
            resize();
        }
        data[size] = value;
        size++;
    }

    T &operator[](int index) {
        return data[index];
    }

    const T &operator[](int index) const {
        return data[index];
    }

    int getSize() const { return size; }
    int getCapacity() const { return capacity; }
};

#endif
```

> ⚠️ **模板类的 .cpp 分离问题**：普通的类可以把声明放 `.h`、实现放 `.cpp`。但对于**模板类**，编译器需要看到完整实现才能生成对应类型的代码。因此，模板类的**全部实现通常都放在 `.h` 头文件中**（或者 `.h` + `.tpp` 并在 `.h` 末尾 `#include`）。上面的例子我们就是全部写在了 `.h` 里——这是模板类的工程惯例。

### 6.4 使用模板类

```cpp
// main.cpp —— 一份代码，三种类型
#include "Vector.h"
#include <iostream>
#include <string>
using namespace std;

int main() {
    // ===== int 版本 =====
    Vector<int> intVec(2);
    intVec.pushBack(10);
    intVec.pushBack(20);
    intVec.pushBack(30);
    cout << "\n=== Vector<int> ===" << endl;
    for (int i = 0; i < intVec.getSize(); i++) {
        cout << intVec[i] << " ";
    }
    cout << endl;

    // ===== double 版本 =====
    Vector<double> doubleVec;
    doubleVec.pushBack(3.14);
    doubleVec.pushBack(2.71);
    doubleVec.pushBack(1.41);
    cout << "\n=== Vector<double> ===" << endl;
    for (int i = 0; i < doubleVec.getSize(); i++) {
        cout << doubleVec[i] << " ";
    }
    cout << endl;

    // ===== string 版本 =====
    Vector<string> strVec;
    strVec.pushBack("Hello");
    strVec.pushBack("InfiniTensor");
    strVec.pushBack("训练营");
    cout << "\n=== Vector<string> ===" << endl;
    for (int i = 0; i < strVec.getSize(); i++) {
        cout << strVec[i] << " ";
    }
    cout << endl;

    return 0;
}
```

编译运行：

```bash
g++ main.cpp -o vector_demo
./vector_demo
```

输出：

```
⚡ Vector<i> 扩容：2 → 4

=== Vector<int> ===
10 20 30

=== Vector<double> ===
3.14 2.71 1.41

=== Vector<string> ===
Hello InfiniTensor 训练营
```

> 🎯 **模板在 AI 框架中的意义**：在 InfiniTensor 中，张量可以是 `float`、`double`、`int8_t`（量化推理）、`bfloat16` 等各种数据类型。如果不用模板，每种类型都要重写一套张量运算代码——维护成本不可想象。**模板让同一套逻辑无缝适配所有数据类型**，这正是现代 C++ 大模型框架的基石。

---

## 🛠️ 第七部分：综合实战——手写完整 Vector 并通过 Valgrind 检测

前面的每个部分都是积木，现在我们把它们全部组装起来，写出一个**功能完整、内存安全、通过 Valgrind 检测**的模板类 Vector。

### 7.1 完整代码

创建文件 `MyVector.h`：

```cpp
#ifndef MYVECTOR_H
#define MYVECTOR_H

#include <iostream>
#include <stdexcept>     // std::out_of_range 异常
using namespace std;

// ===== 模板类 Vector =====
// 功能：动态扩容数组，支持任意数据类型
// 特性：
//   1. 自动扩容（容量翻倍策略）
//   2. pushBack 尾部追加
//   3. operator[] 下标访问（带边界检查的 at()）
//   4. 拷贝构造与赋值运算符（深拷贝）
//   5. 析构时自动释放堆内存
template <typename T>
class Vector {
private:
    T *data;            // 指向堆上的动态数组
    int size;           // 当前元素数量
    int capacity;       // 当前可容纳的最大元素数（容量）

    // ========== 私有方法：扩容 ==========
    void resize() {
        int newCapacity = (capacity == 0) ? 1 : capacity * 2;
        T *newData = new T[newCapacity];

        // 拷贝旧数据到新数组
        for (int i = 0; i < size; i++) {
            newData[i] = data[i];
        }

        // 释放旧数组，指针转向新数组
        delete[] data;
        data = newData;
        capacity = newCapacity;
    }

public:
    // ========== 构造函数 ==========
    // 默认构造：初始容量为 0（惰性分配，首次 pushBack 时扩容到 1）
    Vector() : data(nullptr), size(0), capacity(0) {
        // 什么也不分配，等 pushBack 时再分配
    }

    // 带初始容量的构造
    Vector(int initialCapacity) : size(0), capacity(initialCapacity) {
        data = new T[capacity];
    }

    // ========== 拷贝构造函数（深拷贝） ==========
    Vector(const Vector &other) : size(other.size), capacity(other.capacity) {
        data = new T[capacity];
        for (int i = 0; i < size; i++) {
            data[i] = other.data[i];
        }
    }

    // ========== 赋值运算符（深拷贝） ==========
    Vector &operator=(const Vector &other) {
        if (this == &other) {        // 防止自我赋值
            return *this;
        }

        delete[] data;               // 释放旧资源

        size = other.size;
        capacity = other.capacity;
        data = new T[capacity];
        for (int i = 0; i < size; i++) {
            data[i] = other.data[i];
        }

        return *this;
    }

    // ========== 析构函数 ==========
    ~Vector() {
        delete[] data;
        data = nullptr;
    }

    // ========== 在末尾添加元素 ==========
    void pushBack(const T &value) {
        if (size >= capacity) {
            resize();
        }
        data[size] = value;
        size++;
    }

    // ========== 移除末尾元素 ==========
    void popBack() {
        if (size > 0) {
            size--;   // 不需要真正删除，size 减小后该位置「逻辑不可见」
        }
    }

    // ========== 下标访问（不检查边界，追求性能） ==========
    T &operator[](int index) {
        return data[index];
    }

    const T &operator[](int index) const {
        return data[index];
    }

    // ========== 带边界检查的安全访问 ==========
    T &at(int index) {
        if (index < 0 || index >= size) {
            throw out_of_range("Vector::at() —— 下标越界！");
        }
        return data[index];
    }

    // ========== 清空 Vector ==========
    void clear() {
        size = 0;   // 逻辑清空，不释放底层内存（下次 pushBack 可以复用）
    }

    // ========== 属性获取 ==========
    int getSize() const { return size; }
    int getCapacity() const { return capacity; }
    bool isEmpty() const { return size == 0; }

    // ========== 打印 Vector 信息（调试用） ==========
    void printInfo() const {
        cout << "Vector: size=" << size
             << ", capacity=" << capacity
             << ", elementSize=" << sizeof(T)
             << ", totalMemory=" << (capacity * sizeof(T)) << " bytes" << endl;
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

#endif   // MYVECTOR_H
```

创建测试文件 `test_vector.cpp`：

```cpp
#include "MyVector.h"
#include <iostream>
#include <string>
using namespace std;

int main() {
    cout << "╔══════════════════════════════════════╗" << endl;
    cout << "║  简易 Vector 动态数组 —— 综合测试    ║" << endl;
    cout << "╚══════════════════════════════════════╝" << endl;

    // ===== 测试 1：基本操作与自动扩容 =====
    cout << "\n===== 测试 1：int 类型 + 自动扩容 =====" << endl;
    Vector<int> intVec;
    cout << "初始状态 —— ";
    intVec.printInfo();

    for (int i = 1; i <= 10; i++) {
        intVec.pushBack(i * 10);
    }
    cout << "添加 10 个元素后 —— ";
    intVec.printInfo();
    cout << "内容: ";
    intVec.printAll();
    // 输出：[10, 20, 30, 40, 50, 60, 70, 80, 90, 100]

    // ===== 测试 2：下标访问与修改 =====
    cout << "\n===== 测试 2：下标访问 =====" << endl;
    cout << "intVec[0] = " << intVec[0] << endl;        // 10
    cout << "intVec[5] = " << intVec[5] << endl;        // 60
    intVec[0] = 999;
    cout << "修改后 intVec[0] = " << intVec[0] << endl;  // 999

    // ===== 测试 3：popBack 和 isEmpty =====
    cout << "\n===== 测试 3：popBack =====" << endl;
    cout << "弹出前 size = " << intVec.getSize() << endl;   // 10
    intVec.popBack();
    intVec.popBack();
    intVec.popBack();
    cout << "弹出 3 个后 size = " << intVec.getSize() << endl;  // 7
    cout << "内容: ";
    intVec.printAll();
    // 输出：[999, 20, 30, 40, 50, 60, 70]

    // ===== 测试 4：double 类型 Vector =====
    cout << "\n===== 测试 4：Vector<double> =====" << endl;
    Vector<double> doubleVec(2);
    doubleVec.pushBack(3.14159);
    doubleVec.pushBack(2.71828);
    doubleVec.pushBack(1.41421);
    cout << "内容: ";
    doubleVec.printAll();
    // 输出：[3.14159, 2.71828, 1.41421]

    // ===== 测试 5：string 类型 Vector =====
    cout << "\n===== 测试 5：Vector<string> =====" << endl;
    Vector<string> strVec;
    strVec.pushBack("InfiniTensor");
    strVec.pushBack("大模型");
    strVec.pushBack("训练营");
    strVec.pushBack("2026");
    cout << "内容: ";
    strVec.printAll();
    // 输出：[InfiniTensor, 大模型, 训练营, 2026]

    // ===== 测试 6：拷贝构造 =====
    cout << "\n===== 测试 6：拷贝构造（深拷贝验证） =====" << endl;
    Vector<int> copyVec(intVec);    // 拷贝构造
    intVec[0] = 0;                   // 修改原 Vector
    cout << "原 Vector[0] = " << intVec[0] << endl;     // 0
    cout << "拷贝 Vector[0] = " << copyVec[0] << endl;  // 999（未被影响！）
    cout << "✅ 深拷贝验证通过：修改原对象不影响拷贝" << endl;

    // ===== 测试 7：赋值运算符 =====
    cout << "\n===== 测试 7：赋值运算符 =====" << endl;
    Vector<int> assignVec;
    assignVec = copyVec;             // 赋值
    copyVec[1] = -1;
    cout << "被赋值方[1] = " << copyVec[1] << endl;      // -1
    cout << "赋值结果方[1] = " << assignVec[1] << endl;  // 20（未被影响！）
    cout << "✅ 赋值深拷贝验证通过" << endl;

    // ===== 测试 8：clear 和 边界检查 =====
    cout << "\n===== 测试 8：clear 与 at() 边界检查 =====" << endl;
    strVec.clear();
    cout << "clear 后 isEmpty = " << (strVec.isEmpty() ? "true" : "false") << endl;
    try {
        strVec.at(0);    // 应该抛出异常
    } catch (const out_of_range &e) {
        cout << "✅ 边界检查捕获异常: " << e.what() << endl;
    }

    cout << "\n===== 全部测试通过！ =====" << endl;
    return 0;
}
```

### 7.2 编译与运行

```bash
# 编译（-g 加调试信息，-Wall 开启所有警告）
g++ -g -Wall test_vector.cpp -o test_vector

# 运行
./test_vector
```

### 7.3 Valgrind 内存检测

```bash
# 全面内存检测
valgrind --leak-check=full --show-leak-kinds=all ./test_vector
```

期望看到的关键输出：

```
==12345== HEAP SUMMARY:
==12345==     in use at exit: 0 bytes in 0 blocks
==12345==   total heap usage: X allocs, X frees, Y bytes allocated
==12345==
==12345== All heap blocks were freed -- no leaks are possible
```

如果出现 `definitely lost`、`indirectly lost` 或 `still reachable`，说明代码里有内存泄漏或者没释放的资源，需要回头检查析构函数、赋值运算符、拷贝构造函数是否正确释放了所有 `new[]` 出来的内存。

---

## 📝 总结与系列导航

这篇文章从动态内存出发，一路打通了 C++ 面向对象、模板编程、运算符重载、工程分离编译，最后从零手写了一个通过 Valgrind 内存检测的通用 Vector 动态数组。来梳理一下核心收获：

| 板块 | 核心收获 |
|------|---------|
| **动态内存管理** | 理解了栈 vs 堆的本质区别，掌握 `new` / `delete`、`new[]` / `delete[]` 的配对使用 |
| **Valgrind 内存检测** | 学会用 `valgrind --leak-check=full` 排查内存泄漏，能精确到源码行号 |
| **类与面向对象** | 掌握 class 定义、public/private 访问控制、构造函数与析构函数的 RAII 设计思想 |
| **运算符重载** | 理解 `operator[]` 和 `operator=` 的实现方式与深拷贝的必要性 |
| **文件分离编译** | 掌握 `.h` / `.cpp` 分离规范、`#ifndef` 头文件守卫、多文件编译命令 |
| **C++ 模板** | 理解 `template <typename T>` 的泛型编程思想，模板类让同一份代码适配所有数据类型 |
| **综合实战：Vector** | 从零实现了动态扩容、深拷贝、边界检查、多类型支持的完整 Vector，并通过 Valgrind 检测 |

这些知识不是孤立的——它们是理解 InfiniTensor 等大型 C++ AI 框架架构的**底层基石**。张量（Tensor）本质上就是一个多维的模板类 Vector；内存池管理依赖构造/析构的 RAII 设计；模板编程让同一套算子代码适配 float、double、int8 等所有数据类型。

从 WSL 环境搭建 → C++ 基础语法 → Git 与进阶语法 → 面向对象与模板，我们已经完成了训练营 C++ 模块的核心知识储备。下一篇将进入 **C++ STL 标准库与高性能容器**，学习 `std::vector`、`std::map`、`std::unordered_map` 等工业级容器的使用与性能对比，以及算法复杂度的基础概念。

> 🧱 内存、对象、模板——这三块 C++ 工程基石已经就位。下一站，我们站在巨人的肩膀上，看工业级标准库如何优雅地解决我们手写 Vector 时遇到的所有问题。继续冲！

📖 全系列文章入口：
- 个人博客：[https://liuyuqi-qiqiqi.github.io](https://liuyuqi-qiqiqi.github.io)
- 源码与博客仓库：[https://github.com/liuyuqi-qiqiqi/liuyuqi-qiqiqi.github.io](https://github.com/liuyuqi-qiqiqi/liuyuqi-qiqiqi.github.io)
- 更多训练营系列教程持续更新中，欢迎关注！

---

**个人博客地址**：https://liuyuqi-qiqiqi.github.io
**个人GitHub仓库地址**：https://github.com/liuyuqi-qiqiqi/liuyuqi-qiqiqi.github.io
