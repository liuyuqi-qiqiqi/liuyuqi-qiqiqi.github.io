---
title: 2026年夏季InfiniTensor训练营｜Git版本管理、代码格式化与C++进阶语法
date: 2026-07-08 15:30:00
tags: 2026训练营,InfiniTensor,C++,Git,clang-format,指针,结构体
categories: 2026夏季InfiniTensor训练营
cover:
description: 本篇学习Git版本控制工具使用、clang-format代码统一格式化规范，掌握C++常量关键字、静态数组、指针、结构体核心知识点，夯实大模型开发底层C++功底。
---

# 📦 2026年夏季InfiniTensor训练营｜Git版本管理、代码格式化与C++进阶语法

[上一篇文章](https://liuyuqi-qiqiqi.github.io/2026/07/05/2026-InfiniTensor-training-camp-Cpp-calculator/) 我们搞定了 C++ 基础语法，还写了一个能在终端跑的简易计算器。变量、循环、函数传参这些基本功都过了一遍——**但真正的工程开发不止是写代码**。

在大模型 AI 系统开发中，你需要和团队成员协作、你的代码需要长期维护、你的工程需要规范一致的风格。今天这篇文章，我们就来补上这三块关键拼图：**Git 版本管理**、**clang-format 代码格式化**，以及深入 **C++ 常量、数组、指针、结构体**等进阶语法。内容扎实、干货密集，准备好就出发！

---

## 🔧 第一部分：Git 版本管理工具基础使用

### 1.1 Git 是什么？为什么大模型开发离不开它？

想象一个场景：你花了两周写了一个推理优化模块，有一天改着改着程序崩了，却再也想不起改了什么——**Git 就是用来解决这种灾难的**。

**Git** 是目前全球最流行的分布式版本控制系统。用大白话说，它有两项核心能力：

1. **时间回溯**：每次你完成一个阶段性工作，用 Git 做一个 `commit`（快照），以后任何时候都能「穿越」回这个快照，修改记录一目了然
2. **多人协作**：你和队友各自在独立分支上干活，互不干扰，写完通过 `merge`（合并）把大家的工作拼在一起，不会互相覆盖

在 InfiniTensor 这类大型开源 AI 项目中，几乎每一行代码提交都要通过 Git 来管理——它是软件工程的**基础设施**。

### 1.2 WSL Ubuntu 下安装 Git

[上一篇](https://liuyuqi-qiqiqi.github.io/2026/07/04/2026-InfiniTensor-training-camp-WSL-installation/) 装开发工具链的时候其实已经顺手装了 Git。如果你的环境还没有，一条命令搞定：

```bash
# 安装 Git
sudo apt install -y git

# 确认安装成功
git --version
# 输出示例：git version 2.43.0
```

### 1.3 全局配置用户名和邮箱

Git 要求每次 `commit` 必须带上作者信息，这样团队协作时能知道谁提交了哪些代码。先做一次性配置：

```bash
# 配置用户名（替换成你自己的）
git config --global user.name "liuyuqi"

# 配置邮箱（替换成你自己的）
git config --global user.email "your_email@example.com"

# 查看配置是否生效
git config --global --list
```

> 💡 `--global` 参数表示全局生效，本机所有 Git 仓库都会用这个配置。如果想单独给某个项目设置不同的用户名，去掉 `--global` 在项目目录里执行就行。

### 1.4 核心基础命令实操

下面这六条命令，覆盖了日常开发 90% 的使用场景。我们来用一个小 C++ 项目从头演练一遍：

```bash
# ① 初始化仓库：让当前目录被 Git 管理
mkdir demo-git && cd demo-git
git init
# 输出：Initialized empty Git repository in ~/demo-git/.git/
# → 多了一个隐藏的 .git 目录，Git 的所有版本数据都存在里面

# ② 往文件夹里写一个 C++ 文件
echo '#include <iostream>
using namespace std;

int main() {
    cout << "Git Demo Project" << endl;
    return 0;
}' > main.cpp

# ③ 查看状态：看看哪些文件改了、哪些还没被 Git 追踪
git status
# 输出会显示 main.cpp 是 "Untracked files"（未被追踪的红色文件）

# ④ 添加暂存区：告诉 Git「这个文件我要追踪」
git add main.cpp
# 再用 git status 看一下，main.cpp 变绿了，说明进入了暂存区

# ⑤ 提交版本：把暂存区的改动正式记录为一个版本
git commit -m "init: 创建主程序 main.cpp"
# -m 后面跟的是提交说明，描述这次改了什么
# 输出：1 file changed, 7 insertions(+) 等统计信息

# ⑥ 查看提交历史
git log --oneline
# 输出类似：
# a1b2c3d init: 创建主程序 main.cpp
# --oneline 让每条 commit 压缩成一行，更清爽
```

用一张表来总结这六条核心命令：

| 命令 | 作用 | 类比 |
|------|------|------|
| `git init` | 初始化本地仓库 | 新建一个「时光机」 |
| `git status` | 查看工作区状态 | 看看哪些文件改过了 |
| `git add <file>` | 把文件添加到暂存区 | 把想保存的东西放进购物车 |
| `git commit -m "说明"` | 提交一个版本快照 | 结账，留下一张收据 |
| `git log` | 查看提交历史 | 翻之前的收据记录 |
| `git diff` | 查看文件具体改了什么 | 逐行对比修改内容 |

### 1.5 继续修改——体验 Git 的版本回溯

Git 的强大之处在于**记录每一次改动**，我们来改一下刚才的代码，感受版本追踪的魅力：

```bash
# 修改 main.cpp，增加一句话
echo '
#include <iostream>
using namespace std;

int main() {
    cout << "Git Demo Project v2.0" << endl;
    cout << "Now tracking changes!" << endl;
    return 0;
}' > main.cpp

# 查看改动细节
git diff main.cpp
# 绿色 + 号行是新增的，红色 - 号行是删除的

# 再次提交
git add main.cpp
git commit -m "feat: 增加第二行输出，升级为 v2.0"

# 查看两次提交的记录
git log --oneline
# 现在应该有两条记录，最新的在最上面
```

> 🎯 **养成好习惯**：每完成一个小功能点就 `commit` 一次，而不是攒几百行代码再统一提交。这样出了 bug 能精确定位到具体哪次改动引入的。

---

## 🎨 第二部分：代码风格规范与 clang-format 格式化工具

### 2.1 为什么统一代码风格在大模型项目中至关重要？

先看两段功能完全一样、但风格迥异的代码：

```cpp
// 风格 A：空格乱七八糟
int add(int a,int b){
return a+b;
}

// 风格 B：格式规整统一
int add(int a, int b) {
    return a + b;
}
```

两段代码编译器都能通过，但在团队协作中，风格不统一是**灾难级问题**：

1. **Code Review 效率低**：Review 的时候大量精力浪费在「这里的缩进是不是多了一个空格」这种无意义争论上，而不是去审查逻辑是否正确
2. **Git Diff 噪音大**：不同风格的提交混杂在一起，看改动记录时满屏都是空格和换行差异，真正的逻辑修改反而被淹没
3. **Bug 风险高**：不规范的格式容易掩盖代码结构问题——比如大括号位置不统一，可能让你误以为某行代码在循环内，其实不在

对于 InfiniTensor 这类动辄几十万行的大模型底层框架，**统一代码风格是铁律**。训练营要求使用 **clang-format** 来做自动格式化。

### 2.2 WSL 下安装 clang-format

```bash
# 安装 clang-format
sudo apt install -y clang-format

# 验证安装
clang-format --version
# 输出示例：clang-format version 18.1.3
```

### 2.3 生成格式化配置文件

`clang-format` 需要一个 `.clang-format` 配置文件来告诉它你想要的风格。在项目根目录下生成：

```bash
# 以 LLVM 风格为模板，生成自定义配置文件
# LLVM 是 clang-format 内置的几种基础风格之一
clang-format -style=llvm -dump-config > .clang-format
```

这会生成一个包含几十项配置参数的文件。对于训练营的 C++ 代码，建议重点调整下面几项（用 Vim 打开 `.clang-format` 修改）：

```yaml
# 推荐的训练营 C++ 规范配置（关键项）
BasedOnStyle: LLVM          # 基于 LLVM 风格
IndentWidth: 4              # 缩进 4 个空格
TabWidth: 4                 # Tab 宽度为 4
UseTab: Never               # 不使用 Tab，全部空格
ColumnLimit: 100            # 每行最多 100 个字符
BreakBeforeBraces: Attach   # 大括号紧跟在语句后面
AllowShortFunctionsOnASingleLine: None   # 函数不能写在同一行
SpaceBeforeParens: ControlStatements     # 控制语句关键字后留空格
```

### 2.4 一键格式化

配置好之后，一行命令就能把整个目录的 C++ 代码全部格式化：

```bash
# 格式化当前目录下所有 .cpp 和 .h 文件
clang-format -i *.cpp *.h

# -i 表示 in-place，直接覆盖原文件
# 不加 -i 则只把格式化结果打印到终端，不修改文件
```

来一个实战对比——先写一段故意写乱的代码：

```cpp
// 格式化前：messy.cpp
#include <iostream>
using namespace std;
int main(){
int x=10;
if(x>5){
cout<<"x is greater than 5"<<endl;}
return 0;
}
```

```bash
# 一键格式化
clang-format -i messy.cpp
```

```cpp
// 格式化后：messy.cpp
#include <iostream>
using namespace std;

int main() {
    int x = 10;
    if (x > 5) {
        cout << "x is greater than 5" << endl;
    }
    return 0;
}
```

瞬间清爽！这就是自动格式化工具的力量——**把风格问题交给工具，把精力留给逻辑问题**。

### 2.5 书写习惯规范小结

除了用工具自动格式化，日常写代码时养成这些习惯会让你的代码更易读：

| 规范项 | 推荐做法 | 避免做法 |
|--------|---------|---------|
| **缩进** | 4 个空格 | Tab 和空格混用 |
| **大括号** | 跟在语句后，独占一行 `if (x) {` | 随意放置 |
| **运算符空格** | `a + b`，左右各留空格 | `a+b` 挤在一起 |
| **变量命名** | `studentCount`（驼峰）或 `student_count`（下划线），选一种并坚持 | 混用、拼音英文混杂 |
| **单行长度** | 不超过 100 字符 | 一条语句写 200 字符 |
| **注释** | `// 解释这段代码为什么这么做` | 无注释或注释只复述代码 |

---

## 🧠 第三部分：C++ 核心进阶知识点

前两篇文章我们学习了变量、控制流、函数等基础语法。这一节我们将进入几个**处于基础到进阶之间的关键知识点**——它们是理解后续面向对象编程、内存管理、数据结构的必备前置。

### 3.1 常量关键字 `const`：给数据加一把「只读锁」

#### 为什么需要 const？

程序里有很多数据，我们只希望它**被读取**，不希望它**被意外修改**。比如圆周率 π、程序中约定的最大数组长度、配置文件里的固定参数——这些值一旦写了就不该变。

`const` 就是 C++ 提供的「只读锁」：声明为 `const` 的变量，编译器会阻止任何试图修改它的操作。

#### 基本用法

```cpp
#include <iostream>
using namespace std;

int main() {
    // const 常量：值一旦初始化就不能再改
    const double PI = 3.1415926;
    const int MAX_SIZE = 100;
    const string APP_NAME = "InfiniTensor";

    // PI = 3.14;   // ❌ 编译报错！const 变量不能被修改

    double radius = 5.0;
    double area = PI * radius * radius;   // ✅ 读取是完全没问题的

    cout << "圆的面积: " << area << endl;
    // 输出：圆的面积: 78.5398

    return 0;
}
```

#### const 的三种典型使用场景

```cpp
#include <iostream>
using namespace std;

// 场景 1：const 常量 —— 替代 #define 宏，类型更安全
const int BUF_SIZE = 256;

// 场景 2：const 修饰函数参数 —— 承诺函数内部不会修改传入的数据
void printArray(const int arr[], int size) {
    // arr[0] = 999;  // ❌ 编译报错！const 约束下不能修改
    for (int i = 0; i < size; i++) {
        cout << arr[i] << " ";
    }
    cout << endl;
}

// 场景 3：const 引用 —— 传大对象时不复制，又保证原数据不被修改
void displayInfo(const string &info) {
    cout << "信息: " << info << endl;
    // info = "hack";  // ❌ 编译报错！
}

int main() {
    int numbers[] = {1, 2, 3, 4, 5};
    printArray(numbers, 5);       // 输出：1 2 3 4 5

    string msg = "Hello InfiniTensor";
    displayInfo(msg);             // 输出：信息: Hello InfiniTensor

    return 0;
}
```

> 🎯 **什么时候用 const？** 一个简单的判断标准：如果你不打算让变量被修改，就加上 `const`。这不仅是给自己看的保险，也是给其他读你代码的人发的信号——「这个值别动」。

### 3.2 静态数组：批量管理同类型数据

数组是一段**连续的**内存空间，用来存放相同类型的多个数据。和逐个声明变量相比，数组更简洁、更方便批量操作。

#### 定义与初始化

```cpp
#include <iostream>
using namespace std;

int main() {
    // 方式 1：定义时指定大小，然后逐个赋值
    int scores[5];
    scores[0] = 88;
    scores[1] = 92;
    scores[2] = 75;
    scores[3] = 85;
    scores[4] = 96;

    // 方式 2：定义时直接初始化（推荐）
    int scores2[5] = {88, 92, 75, 85, 96};

    // 方式 3：不写大小，编译器自动推断
    int scores3[] = {88, 92, 75, 85, 96};   // 自动推断长度为 5

    // 方式 4：部分初始化，其余自动填 0
    int scores4[5] = {88, 92};   // 等价于 {88, 92, 0, 0, 0}

    return 0;
}
```

#### 遍历数组

```cpp
#include <iostream>
using namespace std;

int main() {
    int scores[] = {88, 92, 75, 85, 96};
    int size = 5;

    // 方式 1：经典 for 循环 —— 通过下标访问每个元素
    cout << "=== 学生成绩列表 ===" << endl;
    for (int i = 0; i < size; i++) {
        cout << "第 " << (i + 1) << " 位学生: " << scores[i] << " 分" << endl;
    }

    // 方式 2：范围 for 循环（C++11 起支持）—— 更简洁
    cout << "\n=== 范围 for 遍历 ===" << endl;
    for (int s : scores) {
        cout << s << " ";
    }
    cout << endl;

    // 方式 3：计算总和 —— 数组的经典实际应用
    int sum = 0;
    for (int i = 0; i < size; i++) {
        sum += scores[i];
    }
    double average = static_cast<double>(sum) / size;
    cout << "平均分: " << average << endl;
    // 输出：平均分: 87.2

    return 0;
}
```

#### 数组使用注意事项

| 注意点 | 说明 |
|--------|------|
| **下标从 0 开始** | `arr[0]` 是第一个元素，`arr[n-1]` 是最后一个 |
| **不可越界访问** | `arr[5]` 对一个长度为 5 的数组来说是非法的，但 C++ 不会报错——会读到内存里的垃圾值，这是隐蔽的 bug 来源 |
| **大小必须是常量** | 静态数组的大小在编译时就要确定，不能是运行时输入的变量 |
| **数组名即首地址** | `arr` 本身是一个指向数组第一个元素的指针（这个我们马上在 3.3 节展开讲） |

### 3.3 指针：C++ 最硬核的概念，没有之一

如果你问 C++ 学习者「最难的概念是什么」，十个人里九个会回答——**指针**。但指针并不可怕，它只是帮你直接操作内存地址的工具。搞定指针，你在 C/C++ 开发上的天花板会被大幅抬高。

#### 什么是指针？

每个变量都存储在内存的某一个「房间」里，这个房间有编号——就是**内存地址**。指针是一个特殊的变量，它不存具体的数据，而是**存另一个变量的地址**。

用生活类比：变量是一个**箱子**，里面装着数据；指针是一张**纸条**，上面写着箱子在哪里。

#### 取地址 `&` 和解引用 `*`

```cpp
#include <iostream>
using namespace std;

int main() {
    int num = 42;            // 一个普通的 int 变量
    int *p = &num;           // p 是指针，存的是 num 的地址

    // & 是取地址运算符：&num 得到 num 的地址
    // * 在声明中表示「这是一个指针类型」
    // * 在使用时是解引用运算符：*p 得到 p 所指向的那个变量的值

    cout << "num 的值: " << num << endl;          // 输出：42
    cout << "num 的地址: " << &num << endl;       // 输出类似：0x7fff1234
    cout << "p 里存的内容（即 num 的地址）: " << p << endl;   // 和上一行相同
    cout << "通过 *p 访问 num 的值: " << *p << endl;         // 输出：42

    // 通过指针修改原变量的值
    *p = 100;
    cout << "修改后 num 的值: " << num << endl;   // 输出：100
    // num 也跟着变了！因为 *p 和 num 是同一块内存

    return 0;
}
```

> 💡 **核心理解**：`p` 是地址，`*p` 是那个地址上存放的数据。可以理解成——`p` 是门牌号，`*p` 是开门后看到的真实内容。

#### 指针与数组的关系

前面 3.2 节留了一个悬念——「数组名就是指针」。这其实是 C++ 里一个非常优雅的设计：

```cpp
#include <iostream>
using namespace std;

int main() {
    int arr[] = {10, 20, 30, 40, 50};
    int *p = arr;    // 数组名 arr 就是首元素的地址，等价于 &arr[0]

    cout << "arr[0] = " << arr[0] << endl;   // 10
    cout << "*p     = " << *p << endl;       // 10 —— 指针解引用得到首元素

    // 指针算术：p + 1 指向下一个元素
    cout << "*(p+1) = " << *(p + 1) << endl;  // 20
    cout << "*(p+2) = " << *(p + 2) << endl;  // 30

    // 用指针遍历整个数组
    cout << "=== 指针遍历数组 ===" << endl;
    for (int i = 0; i < 5; i++) {
        cout << *(p + i) << " ";   // 等价于 arr[i]
    }
    cout << endl;
    // 输出：10 20 30 40 50

    return 0;
}
```

> 🎯 数组下标 `arr[i]` 在编译器眼中本质上就是 `*(arr + i)`——这也是为什么 C++ 数组下标从 0 开始的原因：`arr[0]` = `*(arr + 0)`，就是第一个元素。

#### 指针使用注意事项

```cpp
// ⚠️ 三大常见陷阱

// 陷阱 1：未初始化的「野指针」—— 指向哪里不知道，修改它会崩溃
// int *p;       // ❌ 危险！不知道 p 指向哪里
// *p = 10;      // ❌ 程序可能直接崩溃

int *p = nullptr;  // ✅ 初始化为 nullptr，至少保证它不是随机值
// nullptr 是 C++11 引入的空指针关键字，替代旧的 NULL

// 陷阱 2：指向已释放的内存（悬垂指针）
// 指针指向的变量生命周期结束后还去访问它 —— 后面学动态内存时会详细讲

// 陷阱 3：指针加减越界
int arr[3] = {1, 2, 3};
int *ptr = arr;
// cout << *(ptr + 5) << endl;  // ❌ 访问了数组之外的未知内存！
```

> ⚠️ **指针安全三原则**：① 声明时立即初始化；② 解引用前确认不是 `nullptr`；③ 不要越界访问。

### 3.4 结构体 `struct`：打造你自己的复合数据类型

前面学过的 `int`、`double`、`string` 都是语言内置的类型。但现实中很多数据是**复合的**——比如一个学生信息由姓名、学号、成绩组成。`struct` 让你能把这些相关字段打包成一个自定义类型。

#### 定义与基本使用

```cpp
#include <iostream>
#include <string>
using namespace std;

// 定义一个学生结构体
struct Student {
    string name;     // 成员变量：姓名
    int id;          // 成员变量：学号
    double score;    // 成员变量：成绩
};  // ⚠️ 注意这里有个分号，新手容易漏掉！

int main() {
    // 方式 1：先定义再逐个赋值
    Student stu1;
    stu1.name = "张三";
    stu1.id = 2026001;
    stu1.score = 88.5;

    // 方式 2：定义时直接用 {} 初始化（C++11 起支持）
    Student stu2 = {"李四", 2026002, 93.0};

    // 方式 3：指定成员名初始化（C++20 起支持，可读性最好）
    // Student stu3 = {.name = "王五", .id = 2026003, .score = 76.5};

    // 通过 . 运算符访问成员
    cout << "=== 学生信息 ===" << endl;
    cout << "姓名: " << stu1.name << endl;
    cout << "学号: " << stu1.id << endl;
    cout << "成绩: " << stu1.score << endl;

    cout << "\n姓名: " << stu2.name << endl;
    cout << "学号: " << stu2.id << endl;
    cout << "成绩: " << stu2.score << endl;

    return 0;
}
```

#### 结构体数组：批量管理复合数据

```cpp
#include <iostream>
#include <string>
using namespace std;

struct Student {
    string name;
    int id;
    double score;
};

int main() {
    // 结构体数组：三个学生
    Student class1[3] = {
        {"张三", 2026001, 88.5},
        {"李四", 2026002, 93.0},
        {"王五", 2026003, 76.5}
    };

    // 遍历结构体数组
    double totalScore = 0;
    cout << "=== 班级成绩表 ===" << endl;
    for (int i = 0; i < 3; i++) {
        cout << "学号: " << class1[i].id
             << "  姓名: " << class1[i].name
             << "  成绩: " << class1[i].score << endl;
        totalScore += class1[i].score;
    }

    double average = totalScore / 3;
    cout << "班级平均分: " << average << endl;
    // 输出：班级平均分: 86

    return 0;
}
```

#### 结构体与指针结合

结构体和指针的结合是 C++ 中非常常见的模式，尤其在处理链表、树等数据结构时：

```cpp
#include <iostream>
#include <string>
using namespace std;

struct Student {
    string name;
    int id;
    double score;
};

int main() {
    Student stu = {"赵六", 2026004, 91.0};
    Student *p = &stu;   // 指向结构体的指针

    // 通过指针访问结构体成员，有两种写法
    // 方式 1：(*p).成员名  —— 先解引用得到结构体，再用 . 访问成员
    cout << "姓名: " << (*p).name << endl;

    // 方式 2：p->成员名    —— 箭头运算符，更常用、更简洁
    cout << "学号: " << p->id << endl;
    cout << "成绩: " << p->score << endl;

    // 通过指针修改成员
    p->score = 95.5;
    cout << "修改后成绩: " << stu.score << endl;   // 输出：95.5

    return 0;
}
```

> 💡 `p->member` 等价于 `(*p).member`。箭头运算符 `->` 是专门为「通过指针访问结构体成员」设计的语法糖，见到它就想到「用指针去指那个成员」。

---

## 🧪 第四部分：综合小练习——学生成绩管理系统

学完上面四个知识点，我们来把它们串起来写一个小项目。这个练习会用到 **struct** 定义数据结构、**const** 保护只读参数、**数组** 存储多条记录、**指针** 遍历数据。

在 WSL 中创建文件 `student_manager.cpp`：

```cpp
#include <iostream>
#include <string>
using namespace std;

// ========== 1. 定义学生结构体 ==========
struct Student {
    string name;
    int id;
    double score;
};

// ========== 2. const 保护 + 打印单个学生信息 ==========
void printStudent(const Student &stu) {
    // stu.score = 100;   // ❌ const 保护下不能修改
    cout << "学号: " << stu.id
         << "  姓名: " << stu.name
         << "  成绩: " << stu.score << endl;
}

// ========== 3. 指针遍历 + 计算平均分 ==========
double calcAverage(const Student *arr, int size) {
    double total = 0;
    for (int i = 0; i < size; i++) {
        total += arr[i].score;         // 等价于 (*(arr + i)).score
    }
    return total / size;
}

// ========== 4. 主程序 ==========
int main() {
    const int CLASS_SIZE = 3;   // const 常量：班级人数固定

    // 结构体数组初始化
    Student classA[CLASS_SIZE] = {
        {"张三", 2026001, 88.5},
        {"李四", 2026002, 93.0},
        {"王五", 2026003, 76.5}
    };

    // 打印成绩表
    cout << "========== 学生成绩管理系统 v1.0 ==========" << endl;
    cout << "共 " << CLASS_SIZE << " 名学生\n" << endl;

    for (int i = 0; i < CLASS_SIZE; i++) {
        cout << "[" << (i + 1) << "] ";
        printStudent(classA[i]);
    }

    // 用指针计算平均分
    Student *ptr = classA;   // 数组名 = 首元素地址
    double avg = calcAverage(ptr, CLASS_SIZE);
    cout << "\n班级平均分: " << avg << endl;

    // 找出最高分（通过指针遍历）
    Student *best = &classA[0];   // 先假设第一个学生最高
    for (int i = 1; i < CLASS_SIZE; i++) {
        if (classA[i].score > best->score) {
            best = &classA[i];    // 更新指针，指向更高分的学生
        }
    }
    cout << "最高分学生: " << best->name
         << " (" << best->score << " 分)" << endl;

    return 0;
}
```

在 WSL 中编译运行：

```bash
# 编译
g++ student_manager.cpp -o student_manager

# 运行
./student_manager
```

运行效果：

```
========== 学生成绩管理系统 v1.0 ==========
共 3 名学生

[1] 学号: 2026001  姓名: 张三  成绩: 88.5
[2] 学号: 2026002  姓名: 李四  成绩: 93
[3] 学号: 2026003  姓名: 王五  成绩: 76.5

班级平均分: 86
最高分学生: 李四 (93 分)
```

#### 代码知识点拆解

这段 60 行的代码用到了本文学到的所有核心知识点：

| 知识点 | 在代码中的体现 | 对应章节 |
|--------|--------------|---------|
| **结构体 `struct`** | 定义 `Student` 类型，包含 name/id/score 三个字段 | 3.4 |
| **结构体数组** | `classA[CLASS_SIZE]` 存放 3 名学生的数据 | 3.4 |
| **`const` 常量** | `CLASS_SIZE` 固定为 3，`printStudent` 参数用 `const &` | 3.1 |
| **指针** | `Student *ptr` 指向数组，`Student *best` 跟踪最高分 | 3.3 |
| **箭头运算符 `->`** | `best->name`、`best->score` 通过指针访问成员 | 3.4 |
| **函数封装** | `printStudent()` 和 `calcAverage()` 各司其职 | 上一篇 3.1 |

---

## ✅ 学习成果验证

完成上面的综合练习后，尝试独立完成下面三个扩展任务，检验掌握程度：

```bash
# 扩展 1：给 student_manager.cpp 增加新功能 —— 查找指定学号的学生
# 提示：遍历数组，用 if 判断 classA[i].id == targetId

# 扩展 2：创建 Git 仓库，把 student_manager.cpp 用版本管理起来
# 提示：git init → git add → git commit，然后用 git log 查看提交记录

# 扩展 3：用 clang-format 格式化你自己的 student_manager.cpp
# 提示：先 cd 到文件所在目录，然后 clang-format -i student_manager.cpp
```

如果这三个扩展都能独立完成，说明你对 Git、clang-format、C++ 进阶语法的掌握已经到位了！

---

## 📝 总结与系列导航

这篇内容量比较大，我们来梳理一下学到了什么：

| 板块 | 核心收获 |
|------|---------|
| **Git 版本管理** | 掌握了 `init` → `add` → `commit` → `log` 核心工作流，代码从此有了「时光机」 |
| **clang-format 格式化** | 学会了一键统一代码风格，告别手动调空格缩进的噩梦 |
| **const 常量** | 理解了「只读锁」的意义，知道三种典型使用场景 |
| **静态数组** | 会定义、初始化、遍历数组，清楚下标从 0 开始 |
| **指针** | 拿下 C++ 最硬核的概念——取地址 `&`、解引用 `*`、指针与数组的关系 |
| **结构体** | 能自定义复合数据类型，结合指针用 `->` 访问成员 |

从 WSL 环境搭建 → C++ 基础语法 → 版本管理与进阶语法，我们已经走完了训练营 C++ 模块的前三站。下一篇我们将进入 **C++ 面向对象编程（OOP）**——类（class）、构造函数、封装与继承，这些是理解 InfiniTensor 等大型 C++ 框架架构的核心。

> 🚀 三篇连读，从零到一。WSL 是地基、基本语法是砖块、Git 和进阶语法是脚手架——下一篇我们要开始搭真正的「建筑」了！

📖 全系列文章入口：
- 个人博客：[https://liuyuqi-qiqiqi.github.io](https://liuyuqi-qiqiqi.github.io)
- 源码与博客仓库：[https://github.com/liuyuqi-qiqiqi/liuyuqi-qiqiqi.github.io](https://github.com/liuyuqi-qiqiqi/liuyuqi-qiqiqi.github.io)
