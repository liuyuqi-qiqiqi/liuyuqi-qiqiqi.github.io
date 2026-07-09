---
title: 2026年夏季InfiniTensor训练营｜WSL介绍与安装指南
date: 2026-07-04 16:00:00
tags: 训练营, InfiniTensor, WSL, Linux, C++开发环境
categories: 2026夏季InfiniTensor训练营
cover:
description: 本文为InfiniTensor训练营C++模块前置教程，详细介绍WSL的作用、安装步骤与初始化配置，为后续Linux开发打下基础
---

# 🚀 2026年夏季InfiniTensor训练营｜WSL介绍与安装指南

暑假刚刚开始，我参加了 **2026年夏季InfiniTensor大模型与人工智能系统训练营**，真的非常兴奋！训练营的 C++ 模块需要在 Linux 环境下进行开发和调试，而对于我们大多数使用 Windows 的同学来说，装虚拟机太笨重、双系统又太麻烦——**WSL 就是最优解**。

这篇文章是训练营 C++ 模块的前置教程，我会从零开始，一步步带你完成 WSL 的安装与初始化配置。无论你是第一次接触 Linux 的新手，还是想快速搭个干净环境的同学，跟着走一遍就能搞定！

---

## 🤔 第一部分：WSL 是什么？

### 一句话解释

**WSL（Windows Subsystem for Linux）** 是微软官方提供的 Windows 内置 Linux 子系统。它让你可以在 Windows 上**直接运行原生的 Linux 程序**，不需要装虚拟机、不需要搞双系统、也不用租云服务器。

简单来说：**在 Windows 里打开一个终端，你就拥有了一台 Ubuntu 电脑。**

### WSL 2：目前最好的版本

WSL 有两个版本，当前推荐使用的是 **WSL 2**。它和 WSL 1 比起来，提升不是一点半点：

- **完整的 Linux 内核**：WSL 2 运行在微软定制的真实 Linux 内核上，系统调用兼容性大幅提升，几乎所有 Linux 程序都能正常运行
- **文件系统性能飞跃**：得益于虚拟化磁盘技术，像 `git clone`、`apt install` 这些 I/O 密集型操作，速度比 WSL 1 快了好几倍
- **Docker 原生支持**：可以直接在 WSL 2 中跑 Docker，轻量又高效
- **GPU 加速 & CUDA 支持**：对跑大模型训练和推理来说，这个能力太关键了

### 为什么搞大模型开发必须学 WSL？

结合 InfiniTensor 训练营的实际需求，我总结了这几个理由：

1. **C++ 工具链的天然主场**：GCC、CMake、Make、GDB……这些 C++ 开发必备工具，在 Linux 下的体验远好于 Windows。训练营后续的代码编译、调试全都在 Linux 环境里进行
2. **InfiniTensor 框架以 Linux 为第一开发平台**：训练营提供的代码和工具链，在 Linux 下开箱即用，不用折腾各种兼容问题
3. **本地环境 = 线上环境**：大部分 AI 服务器跑的都是 Linux，WSL 让你本地开发和线上部署的环境保持一致，减少「我本地能跑啊」的尴尬
4. **AI 开源生态的门票**：llama.cpp、TensorRT-LLM、vLLM……几乎所有大模型推理框架都在 Linux 上首发支持。装好 WSL，就等于拿到了开源 AI 世界的入场券

---

## 📋 第二部分：安装前准备

### Windows 版本要求

| 系统版本 | 最低要求 |
|---------|---------|
| Windows 10 | 版本 2004 及以上（内部版本 ≥ 19041） |
| Windows 11 | 所有版本均支持 |

> 💡 **怎么查看自己的系统版本？** 按 `Win + R` → 输入 `winver` → 回车，弹出的窗口里就能看到。

### 硬件要求

- **CPU**：支持二级地址转换（SLAT）——不用担心，近十年的 CPU 基本都支持
- **内存**：建议 **8GB 及以上**（WSL 2 默认会动态占用内存，上限可手动调整，后面会讲）

### 管理员权限说明

安装过程中需要在 **PowerShell（管理员模式）** 下执行命令，因为要启用 Windows 的系统级功能。操作很简单：右键点击开始菜单 → 选择「**Windows PowerShell（管理员）**」或「**终端（管理员）**」就行。

---

## 🔧 第三部分：完整安装步骤

好了，准备工作就绪，下面进入实操环节。每一步都对应我实际操作过的过程，跟着来就行。

### 步骤一：一键安装 WSL

以**管理员身份**打开 PowerShell，粘贴下面这条命令并回车：

```bash
# 一键安装 WSL 2 + Ubuntu（推荐！）
wsl --install
```

这条命令会自动帮你完成五件事：
1. 启用「适用于 Linux 的 Windows 子系统」功能
2. 启用「虚拟机平台」功能
3. 下载并安装最新的 WSL Linux 内核
4. 将 WSL 2 设为默认版本
5. 自动安装 **Ubuntu** 发行版（目前默认是 Ubuntu 24.04 LTS）

> ⚠️ 如果你的 Windows 版本比较旧，`wsl --install` 可能不可用，那就改用下面这两条手动命令：
>
> ```bash
> # 手动方式一：启用 WSL 功能
> dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
>
> # 手动方式二：启用虚拟机平台
> dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
> ```

### 步骤二：重启电脑

安装完成后，PowerShell 会提示你**重启计算机**。保存好手头的工作，然后重启。

```bash
# 或者直接在 PowerShell 里执行重启
Restart-Computer
```

### 步骤三：首次启动 Ubuntu

重启之后，在开始菜单搜索 **"Ubuntu"**，点击打开。第一次启动会花几分钟初始化，终端里会显示：

```
Installing, this may take a few minutes...
Please create a default UNIX user account.
```

耐心等它跑完就好。

### 步骤四：创建你的 Linux 用户

初始化完成后，系统会让你设置用户名和密码：

```bash
# 输入用户名（建议用小写拼音，比如我的）
Enter new UNIX username: liuyuqi

# 输入密码（注意：输入时屏幕完全不会显示任何字符，连 * 都没有！）
New password:

# 再输一遍密码确认
Retype new password:
```

> 🔐 **重点提醒**：Linux 终端里输密码时**什么都不显示**——不会出现 `***` 也不会出现光标移动，这是 Linux 的安全设计。别以为键盘坏了，直接盲打、按回车就好！

密码设置完成后，你会看到类似这样的欢迎界面：

```
Installation successful!
Welcome to Ubuntu 24.04 LTS (GNU/Linux 5.15.167.4-microsoft-standard-WSL2 x86_64)

liuyuqi@DESKTOP:~$
```

看到这个提示符 `$`，就说明——**🎉 WSL 安装成功！**

### 步骤五：确认 WSL 版本号

回到 Windows PowerShell，执行下面这条命令，确认 Ubuntu 跑在 WSL 2 模式下：

```bash
# 查看所有已安装的 WSL 发行版及其版本
wsl -l -v
```

正常输出应该是这样：

```
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

**确保 VERSION 列显示的是 `2`。** 如果不小心显示为 `1`，用这条命令升级：

```bash
# 将 Ubuntu 切换到 WSL 2
wsl --set-version Ubuntu 2
```

---

## ⚙️ 第四部分：安装后基础配置

WSL 装好之后，别急着用。先花五分钟做一下基础配置，后续开发会顺手很多。

### 4.1 更新系统软件包

刚装好的系统，软件包列表和软件本身都比较旧，先全部更新一轮：

```bash
# 第一步：更新软件包索引
sudo apt update

# 第二步：升级所有已安装的软件包（-y 表示自动确认，不用一个个按 Y）
sudo apt upgrade -y
```

> 这两步是 Linux 的基本操作，以后装任何新软件之前都可以先跑一遍，保持系统清爽。

### 4.2 切换到 Linux 家目录

WSL 默认打开时，当前目录可能挂在 Windows 的路径下（`/mnt/c/Users/xxx`）。我们一般都在 Linux 自己的家目录下工作，先切过去：

```bash
# 切换到 Linux 家目录
cd ~

# 看看现在在哪，应该输出 /home/liuyuqi（你的用户名）
pwd
```

每次开终端都要手动 `cd ~` 很烦？一行命令搞定自动跳转：

```bash
# 把 cd ~ 写进启动脚本，以后每次打开终端自动回到家目录
echo "cd ~" >> ~/.bashrc
```

### 4.3 安装 C/C++ 开发工具链（强烈推荐）

为训练营后续的代码编译做好准备，先把常用工具装好：

```bash
# C/C++ 编译全家桶：GCC 编译器 + CMake 构建工具 + Make
sudo apt install -y build-essential gcc g++ cmake make

# Git 版本管理（必备中的必备）
sudo apt install -y git

# 其他顺手工具：curl 网络请求、wget 下载、vim 编辑器
sudo apt install -y curl wget vim
```

### 4.4 限制 WSL 的内存和 CPU（可选但建议配置）

WSL 2 默认会「按需」吃掉系统内存，有时候能占到 80% 以上。如果你的电脑内存不是特别充裕，建议手动给它设个上限。

在 Windows 的用户目录下新建 `.wslconfig` 文件（路径：`C:\Users\你的用户名\.wslconfig`），写入以下内容：

```ini
[wsl2]
memory=4GB      # 限制内存最多用 4GB
processors=2    # 限制最多用 2 个 CPU 核心
swap=2GB        # 交换空间 2GB
```

保存后在 PowerShell 里重启 WSL 让配置生效：

```bash
# 关闭所有 WSL 实例
wsl --shutdown

# 然后重新打开 Ubuntu 即可
```

### 4.5 常见问题速查

装的时候万一碰到问题，别慌，下面这张表覆盖了大部分情况：

| 遇到的问题 | 怎么解决 |
|-----------|---------|
| 提示「请启用虚拟机平台」 | 管理员 PowerShell 执行：`dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart` → 重启电脑 |
| `wsl --install` 报错/命令不存在 | 去 [aka.ms/wsl2kernel](https://aka.ms/wsl2kernel) 手动下载安装 WSL 内核更新包 |
| 安装失败提示 BIOS 虚拟化未开启 | 重启进入 BIOS，找到 **Intel VT-x** 或 **AMD SVM**，设为 **Enabled** |
| WSL 突然连不上网 | PowerShell 执行 `wsl --shutdown`，然后重新打开 Ubuntu 即可 |

---

## ✅ 配置完成验证

全部搞定之后，跑下面这几条命令，确认环境一切正常：

```bash
# 1. 查看 Linux 发行版信息
lsb_release -a

# 2. 查看 GCC 版本（确认 C++ 编译工具链就绪）
gcc --version

# 3. 查看 Git 版本
git --version

# 4. 确认当前用户和家目录
whoami && pwd
```

如果一切正常，你应该能看到类似这样的输出：

```
Distributor ID: Ubuntu
Description:    Ubuntu 24.04 LTS
Release:        24.04
Codename:       noble

gcc (Ubuntu 13.2.0-4ubuntu3) 13.2.0
git version 2.43.0

liuyuqi
/home/liuyuqi
```

每一项都对上了？那么恭喜你——**WSL 开发环境已全部就绪！** 🎉

---

## 📝 后续预告

Linux 环境已经搭好了，下一步就是正式进入 **C++ 基础语法** 的学习。我会从数据类型、控制流、函数、指针这些核心概念讲起，一步一步打好基础，为后续 InfiniTensor 大模型框架的底层开发做好准备。

> 🧱 打好地基，才能盖高楼。WSL 只是第一步，接下来的旅程会更硬核也更有趣，一起冲！💪


