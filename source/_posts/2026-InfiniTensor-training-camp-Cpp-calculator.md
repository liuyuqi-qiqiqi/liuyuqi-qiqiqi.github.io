---
title: 2026年夏季InfiniTensor训练营｜C++基础语法与简易计算器实现
date: 2026-07-05 10:00:00
tags: 训练营, InfiniTensor, C++, Linux开发, 编程入门
categories: 2026夏季InfiniTensor训练营
cover:
description: 本文为InfiniTensor训练营C++模块核心教程，从编译原理、基础语法到函数传参，完整实现一个简易加减法计算器，为后续AI开发打下基础
---

# 🖥️ 2026年夏季InfiniTensor训练营｜C++基础语法与简易计算器实现

[上一篇文章](https://liuyuqi-qiqiqi.github.io/2026/07/04/2026-InfiniTensor-training-camp-WSL-installation/) 我们已经把 WSL 开发环境搭好了，Ubuntu 终端里 GCC、CMake、Git 全部就位。**环境是用来写代码的**——今天这篇文章，我们正式进入 C++ 编程的大门！

我会从最基础的编译原理讲起，逐步覆盖变量、数据类型、控制流、函数传参等核心语法，最后带你**亲手实现一个能在终端运行的简易加减法计算器**。每一行代码都可以在上一篇搭好的 WSL 环境里直接跑，真正做到「看完就能上手」。

---

## 🤔 第一部分：从源代码到可执行程序——C++ 编译原理

在写第一行代码之前，有必要先搞清楚一件事：**我们写的 `.cpp` 文本文件，是怎么变成一个能双击运行的程序？**

### 一个最小 C++ 程序

用 Vim（或者你习惯的任何编辑器）创建一个文件 `hello.cpp`：

```bash
# 在 WSL 终端中创建并编辑文件
vim hello.cpp
```

写入下面这几行代码：

```cpp
#include <iostream>          // 引入输入输出流库
using namespace std;         // 使用标准命名空间

int main() {
    cout << "Hello, InfiniTensor!" << endl;   // 输出一句话
    return 0;                 // 程序正常结束
}
```

保存后，用 `g++` 编译运行：

```bash
# 编译 hello.cpp，生成可执行文件 hello
g++ hello.cpp -o hello

# 运行
./hello
```

终端输出：

```
Hello, InfiniTensor!
```

### 编译到底做了什么？

`g++ hello.cpp -o hello` 这条命令看似简单，实际上背后经历了四个阶段：

| 阶段 | 作用 | 产物 |
|------|------|------|
| **预处理** | 展开 `#include` 头文件、替换 `#define` 宏 | `.i` 预处理文件 |
| **编译** | 将 C++ 代码翻译为汇编语言 | `.s` 汇编文件 |
| **汇编** | 将汇编代码转为机器码（二进制指令） | `.o` 目标文件 |
| **链接** | 把多个 `.o` 文件和库合并成可执行文件 | `hello`（可执行程序） |

> 💡 **通俗理解**：预处理是「把材料备齐」，编译是「翻译成中间语言」，汇编是「转成机器能懂的 0/1」，链接是「把所有零件拼起来」。GCC/G++ 一条命令帮你自动完成全部四步，非常方便。

---

## 📋 第二部分：C++ 基础语法速览

掌握了编译流程，接下来系统性地过一遍最常用的语法。下面每个知识点都会配代码示例，建议边看边在 WSL 里敲一遍。

### 2.1 变量与数据类型

变量是存储数据的容器，C++ 是**强类型语言**，每个变量必须先声明类型才能使用：

```cpp
#include <iostream>
using namespace std;

int main() {
    // 基本数据类型
    int age = 20;              // 整数
    double price = 12.5;       // 双精度浮点数（小数）
    char grade = 'A';          // 单个字符
    bool isPass = true;        // 布尔值（true / false）
    string name = "liuyuqi";   // 字符串（需要 #include <string>）

    cout << "姓名: " << name << ", 年龄: " << age << endl;
    return 0;
}
```

几种最常用类型的对比：

| 类型 | 关键字 | 占用大小 | 示例值 |
|------|--------|---------|--------|
| 整数 | `int` | 4 字节 | `42`, `-10` |
| 浮点数 | `double` | 8 字节 | `3.14`, `-0.5` |
| 字符 | `char` | 1 字节 | `'A'`, `'z'` |
| 布尔 | `bool` | 1 字节 | `true`, `false` |
| 字符串 | `string` | 动态 | `"Hello"` |

### 2.2 输入与输出

C++ 使用 `cin` 和 `cout` 来处理终端的输入输出：

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    string name;
    int age;

    // 输出提示
    cout << "请输入你的名字: ";
    // 接收用户输入
    cin >> name;

    cout << "请输入你的年龄: ";
    cin >> age;

    // 输出结果
    cout << "你好, " << name << "! 你今年 " << age << " 岁。" << endl;

    return 0;
}
```

> 💡 也许你会注意到，输完数据按 Tab 没有任何反应？你需要按**回车键(Enter)**来结束输入。

### 2.3 运算符

C++ 的运算符和数学里的用法基本一致，上手很快：

```cpp
int a = 10, b = 3;

// 算术运算
int sum = a + b;        // 加法 → 13
int diff = a - b;       // 减法 → 7
int prod = a * b;       // 乘法 → 30
int quot = a / b;       // 整数除法 → 3（注意：小数部分被截断！）
int remain = a % b;     // 取模（求余数）→ 1

// 比较运算（结果为 bool 类型）
bool isEqual = (a == b);    // false
bool isGreater = (a > b);   // true
bool isNotEqual = (a != b); // true

// 逻辑运算
bool result = (a > 5) && (b < 5);   // true（与：两边都真才真）
bool result2 = (a > 5) || (b > 5);  // true（或：有一边真就真）
```

### 2.4 控制流：条件判断

程序不能只会「一条路走到黑」，条件判断让它学会做选择：

```cpp
int score = 85;

if (score >= 90) {
    cout << "优秀！" << endl;
} else if (score >= 60) {
    cout << "及格！" << endl;
} else {
    cout << "需要加油！" << endl;
}
// 输出：及格！

// 也可以用 switch（适合多分支等值判断）
char op = '+';
switch (op) {
    case '+': cout << "加法" << endl; break;
    case '-': cout << "减法" << endl; break;
    default:  cout << "未知运算" << endl;
}
```

### 2.5 控制流：循环

重复执行的任务交给循环，C++ 提供三种常用循环：

```cpp
// for 循环：知道循环次数时使用
for (int i = 1; i <= 5; i++) {
    cout << "第 " << i << " 次循环" << endl;
}

// while 循环：循环次数不确定时使用
int count = 0;
while (count < 3) {
    cout << "count = " << count << endl;
    count++;
}

// do-while 循环：至少执行一次
int num;
do {
    cout << "请输入一个正数（输入 0 退出）: ";
    cin >> num;
} while (num != 0);
```

---

## 🔧 第三部分：函数与参数传递

函数是代码复用的基本单元。把一段逻辑封装成一个函数，后面需要的时候直接调用就行，不用复制粘贴。

### 3.1 函数的定义与调用

```cpp
#include <iostream>
using namespace std;

// 函数定义：返回类型 + 函数名 + 参数列表 + 函数体
int add(int x, int y) {
    return x + y;    // 返回两数之和
}

// void 类型：不需要返回值的函数
void printWelcome() {
    cout << "=== 简易计算器 ===" << endl;
}

int main() {
    printWelcome();                    // 调用无参函数

    int result = add(5, 3);           // 调用有参函数
    cout << "5 + 3 = " << result << endl;
    // 输出：5 + 3 = 8

    return 0;
}
```

### 3.2 值传递 vs 引用传递 —— 初学者最容易踩的坑

C++ 函数传参有两种方式，理解它们的区别非常重要：

```cpp
#include <iostream>
using namespace std;

// 值传递：函数内修改不影响外部变量（复制一份进去）
void swapByValue(int a, int b) {
    int temp = a;
    a = b;
    b = temp;
    cout << "值传递内部: a=" << a << ", b=" << b << endl;
    // 输出：a=20, b=10（内部确实交换了）
}

// 引用传递（带 & 符号）：函数内修改直接作用于外部变量
void swapByReference(int &a, int &b) {
    int temp = a;
    a = b;
    b = temp;
}

int main() {
    int x = 10, y = 20;

    swapByValue(x, y);
    cout << "值传递后外部: x=" << x << ", y=" << y << endl;
    // 输出：x=10, y=20 → 没变！因为只交换了副本

    swapByReference(x, y);
    cout << "引用传递后外部: x=" << x << ", y=" << y << endl;
    // 输出：x=20, y=10 → 真正交换了！

    return 0;
}
```

> 🎯 **记忆口诀**：不带 `&` 是「复印件」，改了白改；带 `&` 是「原件」，改了生效。后续我们会看到，大块数据的传递也喜欢用引用，因为它只需传地址，免去了整体拷贝的开销。

---

## ⚙️ 第四部分：实战——简易加减法计算器

前面所有的知识点，最终都是为了「写点有用的东西」。现在我们把变量、输入输出、条件判断、函数全部串起来，实现一个**能在终端交互的计算器**。

### 4.1 功能设计

这个计算器支持两种运算，结构简单但五脏俱全：

1. 用户选择运算类型：加法（+）或减法（-）
2. 用户输入两个数字
3. 程序计算结果并输出
4. 用户可以选择继续计算或退出

### 4.2 完整代码

在 WSL 中创建文件 `calculator.cpp`，写入以下代码：

```cpp
#include <iostream>
using namespace std;

// 加法函数
double add(double a, double b) {
    return a + b;
}

// 减法函数
double subtract(double a, double b) {
    return a - b;
}

// 打印菜单
void showMenu() {
    cout << "======================" << endl;
    cout << "  简易计算器  v1.0" << endl;
    cout << "======================" << endl;
    cout << "  支持运算：" << endl;
    cout << "  +  加法" << endl;
    cout << "  -  减法" << endl;
    cout << "  q  退出程序" << endl;
    cout << "======================" << endl;
}

int main() {
    char op;           // 存储用户选择的运算符
    double num1, num2, result;   // 两个操作数 + 结果
    bool running = true;         // 控制主循环

    showMenu();

    while (running) {
        // 第一步：获取运算符
        cout << "\n请选择运算 (+, -, q=退出): ";
        cin >> op;

        // 第二步：判断是否退出
        if (op == 'q' || op == 'Q') {
            cout << "感谢使用，再见！👋" << endl;
            running = false;
            continue;   // 跳过本次循环剩余代码
        }

        // 第三步：获取两个操作数
        cout << "请输入第一个数字: ";
        cin >> num1;
        cout << "请输入第二个数字: ";
        cin >> num2;

        // 第四步：根据运算符调用对应的函数
        switch (op) {
            case '+':
                result = add(num1, num2);
                cout << "结果: " << num1 << " + " << num2
                     << " = " << result << endl;
                break;

            case '-':
                result = subtract(num1, num2);
                cout << "结果: " << num1 << " - " << num2
                     << " = " << result << endl;
                break;

            default:
                cout << "⚠️  无效的运算符，请重新输入！" << endl;
                break;
        }
    }

    return 0;
}
```

### 4.3 编译与运行

```bash
# 编译 calculator.cpp
g++ calculator.cpp -o calculator

# 运行计算器
./calculator
```

运行效果示例：

```
======================
  简易计算器  v1.0
======================
  支持运算：
  +  加法
  -  减法
  q  退出程序
======================

请选择运算 (+, -, q=退出): +
请输入第一个数字: 15.5
请输入第二个数字: 4.3
结果: 15.5 + 4.3 = 19.8

请选择运算 (+, -, q=退出): -
请输入第一个数字: 100
请输入第二个数字: 37
结果: 100 - 37 = 63

请选择运算 (+, -, q=退出): q
感谢使用，再见！👋
```

### 4.4 代码结构分析

回顾一下这个 60 行的计算器用到了哪些知识点：

| 知识点 | 在代码中的体现 |
|--------|--------------|
| **变量与数据类型** | `double num1, num2, result` 存储操作数和结果 |
| **输入输出** | `cin` 接收运算符和数字，`cout` 输出计算结果 |
| **函数定义与调用** | `add()`、`subtract()`、`showMenu()` 三个函数 |
| **值传递** | `add(double a, double b)` 中的参数按值传入 |
| **条件判断** | `if-else` 判断是否退出，`switch-case` 分发运算符 |
| **循环** | `while (running)` 让计算器持续运行直到用户输入 `q` |

---

## ✅ 学习成果验证

完成以上内容后，建议你独立完成下面三个小练习，检验一下掌握程度：

```bash
# 练习 1：用 g++ 分步查看编译的每个阶段（-E / -S / -c 三个选项）
g++ -E hello.cpp -o hello.i    # 只预处理
g++ -S hello.cpp -o hello.s    # 编译到汇编
g++ -c hello.cpp -o hello.o    # 汇编到目标文件
g++ hello.o -o hello           # 链接成可执行文件

# 练习 2：给计算器增加乘法和除法功能（修改 switch 分支 + 新增函数）
# 提示：除法时记得判断除数不能为 0

# 练习 3：把 add/subtract 函数改为引用传递，思考是否有必要
# 提示：对于基本类型，值传递更简洁；对于大对象，引用传递更高效
```

如果这三个练习都能自己完成，说明你已经掌握了这篇文章的核心内容！

---

## 📝 后续预告

C++ 基础语法和函数已经入门了，下一篇我们将深入 **C++ 的面向对象特性**——类（class）与对象、构造函数、封装与继承。这些不仅是 C++ 区别于 C 语言的核心，也是理解 InfiniTensor 等大型 C++ 框架架构的关键。

> 🧩 语法是砖块，设计模式是图纸。打好语法基础之后，我们就能开始搭真正的「建筑」了。下一篇见！


