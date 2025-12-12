# Unhide 项目源代码分析 / Source Code Analysis

## 项目概述 / Project Overview

**Unhide** 是一个法律取证工具，用于检测被 rootkit、LKM (可加载内核模块) 或其他隐藏技术隐藏的进程和 TCP/UDP 端口。

**Unhide** is a forensic tool to find hidden processes and TCP/UDP ports by rootkits/LKMs or other hiding techniques.

- **项目网站 / Website**: http://www.unhide-forensics.info
- **源码仓库 / Repository**: http://sourceforge.net/projects/unhide/
- **许可证 / License**: GPL v3
- **版本 / Version**: 20240509 (2024年5月9日)
- **主要作者 / Authors**: Yago Jesus & Patrick Gouin

## 项目结构 / Project Structure

### 核心组件 / Core Components

项目包含以下主要可执行程序：

1. **unhide-linux** - Linux >= 2.6 的隐藏进程检测工具
2. **unhide-posix** - 通用 Unix 系统的隐藏进程检测工具 (BSD, Solaris, 旧版Linux)
3. **unhide-tcp** - TCP/UDP 隐藏端口检测工具
4. **unhide_rb** - unhide.rb 的 C 语言移植版本 (轻量级版本)
5. **unhideGui.py** - 图形用户界面 (Python/Tkinter)

### 源代码文件组织 / Source Files Organization

#### unhide-linux 相关文件 (4个C文件)
```
unhide-linux.c              (856行) - 主程序，命令行解析，核心检测函数
unhide-linux-bruteforce.c   (277行) - 暴力破解 PID 空间的实现
unhide-linux-procfs.c       (454行) - 基于 /proc 文件系统的检测方法
unhide-linux-syscall.c      (790行) - 基于系统调用的检测方法
unhide-linux-compound.c     (425行) - 组合检测方法 (quick, reverse)
unhide-linux.h              (166行) - 头文件，定义结构、常量和函数原型
```

#### unhide-tcp 相关文件
```
unhide-tcp.c                (577行) - TCP/UDP 端口检测主程序
unhide-tcp-fast.c           (220行) - 快速扫描实现
unhide-tcp.h                (64行)  - 头文件
```

#### 其他实现
```
unhide-posix.c              (254行) - POSIX 兼容版本
unhide_rb.c                 (665行) - Ruby 版本的 C 移植
```

#### 共享模块
```
unhide-output.c             (208行) - 输出和日志记录功能
unhide-output.h             (50行)  - 输出函数接口
```

#### Python GUI
```
unhideGui.py                (22623字节) - Tkinter 图形界面
ToolTip.py                  - 工具提示支持
```

## 架构设计 / Architecture Design

### unhide-linux 检测架构

Unhide-linux 实现了**六种主要检测技术**，通过对比不同信息源来发现隐藏进程：

#### 1. 进程检测方法 / Process Detection Methods

##### A. /proc 文件系统检测 (unhide-linux-procfs.c)

**checkproc()** - 通过 stat() 扫描 /proc
- 遍历所有可能的 PID (1 到 maxpid)
- 对每个 /proc/[pid] 目录调用 stat()
- 将结果与 ps 命令输出对比
- 在检查前后都进行 stat() 以避免竞态条件

**checkchdir()** - 通过 chdir() 检测
- 使用 chdir() 系统调用尝试进入 /proc/[pid]
- 对比 ps 输出
- 可选 -m 参数进行更多检查，验证线程是否在其领导进程的线程列表中

**checkopendir()** - 通过 opendir() 检测
- 使用 opendir() 打开 /proc/[pid] 目录
- 对比 ps 输出
- 支持 -m 选项进行额外验证

**checkreaddir()** - 通过 readdir() 检测
- 读取 /proc 和 /proc/[pid]/task 目录内容
- 将目录条目与 ps 输出对比

##### B. 系统调用检测 (unhide-linux-syscall.c)

**checkgetpriority()** - 基于 getpriority() 系统调用
- 对每个 PID 调用 getpriority()
- 如果成功但 ps 看不到，则可能是隐藏进程

**checkgetpgid()** - 基于 getpgid() 系统调用
- 获取进程组 ID
- 与 ps 输出对比

**checkgetsid()** - 基于 getsid() 系统调用
- 获取会话 ID
- 与 ps 输出对比

**checksched_getaffinity()** - 基于 sched_getaffinity()
- 检查 CPU 亲和性
- 验证进程存在性

**checksched_getparam()** - 基于 sched_getparam()
- 获取调度参数
- 验证进程

**checksched_getscheduler()** - 基于 sched_getscheduler()
- 获取调度器信息
- 检测隐藏进程

**checksched_rr_get_interval()** - 基于 sched_rr_get_interval()
- 获取实时调度间隔
- 验证进程存在

**checkkill()** - 基于 kill() 系统调用
- 使用 kill(pid, 0) 测试进程存在性 (不实际杀死进程)
- 与 ps 输出对比

**checksysinfo()** / **checksysinfo2()** / **checksysinfo3()** 
- 使用 sysinfo() 系统调用
- 比较系统报告的进程数与 ps 看到的数量
- 多个版本适用于不同的内核 (RT, preempt, latency 补丁)

**checkallnoprocps()** - 纯系统调用对比
- 将不同系统调用的结果互相对比
- 不依赖 /proc 或 ps 输出

##### C. 暴力破解检测 (unhide-linux-bruteforce.c)

**brute()** - PID 空间暴力破解
- 使用两种方法覆盖整个 PID 空间：
  1. **vfork() 方法**：创建子进程占用 PID
  2. **pthread_create() 方法**：创建线程占用 TID
- 标记所有被占用的 PID
- 无法占用的 PID 被检查是否在 ps 中可见
- 支持双重检查 (-d 选项) 减少误报
- 分配两个数组记录两次扫描结果
- 只报告两次都未被占用的 PID

**关键实现细节**：
```c
// 创建线程获取 TID
void *functionThread(void *parametro) {
    tid = (pid_t) syscall(SYS_gettid);
    return(&tid);
}

// 双重检查逻辑
if ((allpids[y] != 0) && 
    ((TRUE == brutesimplecheck) || (allpids2[y] != 0))) {
    // 这个 PID 两次都无法占用，检查是否隐藏
    if(!checkps(allpids[y], PS_PROC | PS_THREAD | PS_MORE)) {
        printbadpid(allpids[y]);
    }
}
```

##### D. 组合检测方法 (unhide-linux-compound.c)

**checkallquick()** - 快速组合检测
- 同时运行多个检测：proc、procfs、sys
- 速度约快 20 倍，但可能有更多误报
- 对每个 PID 执行多个测试：
  - kill(pid, 0)
  - stat(/proc/[pid])
  - opendir(/proc/[pid])
  - chdir(/proc/[pid])
  - getpriority()
  - getpgid()
  - getsid()
  - sched_getaffinity()
  - sched_getparam()
  - sched_getscheduler()
  - sched_rr_get_interval()
- 统计有多少测试找到了进程
- 与 ps 输出对比

**checkallreverse()** - 反向验证
- 验证 ps 看到的所有线程在 procfs 和系统调用中也可见
- 用于检测 rootkit 是否杀死了安全工具并伪造 ps 输出
- 遍历 ps 的输出
- 验证每个进程在 /proc 中可见
- 验证系统调用能访问该进程

#### 2. 测试编号系统 / Test Number System

```c
enum test_num {
    TST_NONE = 0,
    TST_VERSION,
    TST_PROC,           // /proc stat 扫描
    TST_CHDIR,          // chdir() 测试
    TST_OPENDIR,        // opendir() 测试
    TST_READDIR,        // readdir() 测试
    TST_GETPRIO,        // getpriority() 测试
    TST_GETPGID,        // getpgid() 测试
    TST_GETSID,         // getsid() 测试
    TST_GETAFF,         // sched_getaffinity() 测试
    TST_GETPARM,        // sched_getparam() 测试
    TST_GETSCHED,       // sched_getscheduler() 测试
    TST_RR_INT,         // sched_rr_get_interval() 测试
    TST_KILL,           // kill() 测试
    TST_NOPROCPS,       // 纯系统调用对比
    TST_BRUTE,          // 暴力破解
    TST_REVERSE,        // 反向验证
    TST_QUICKONLY,      
    TST_SYS_INFO,       // sysinfo() 测试
    TST_SYS_INFO2,      // sysinfo() 替代版本
    TST_SYS_INFO3,      // sysinfo() 第三版本
    // 元测试
    TST_DIR,            // 所有目录测试
    TST_SYS,            // 所有系统调用测试
    TST_QUICK,          // 快速组合测试
    TST_PROCALL,        // proc + procfs 组合
    MAX_TESTNUM
};
```

### unhide-tcp 架构

检测隐藏的 TCP/UDP 监听端口：

#### 两种主要方法：

**1. 暴力破解所有端口 (Brute Force)**
- 尝试绑定所有 TCP/UDP 端口 (1-65535)
- 将结果与 ss/netstat 输出对比
- 发现未被 ss/netstat 报告的监听端口

**2. 探测未报告端口 (Probe)**
- 分析 ss/netstat 输出找出未使用的端口
- 尝试连接这些"未使用"的端口
- 如果连接成功，说明存在隐藏的监听器

#### 支持的工具：
- **ss** (iproute2) - Linux 默认
- **netstat** (net-tools) - 传统工具
- **fuser** - 显示使用端口的进程
- **lsof** - 列出打开的文件和网络连接
- **sockstat** - FreeBSD 等效工具

### unhide_rb 架构

这是 unhide.rb (Ruby版本) 的 C 语言移植：

**特点**：
- 大约等同于 "unhide-linux quick reverse"
- 但做的测试更少 (少了 kill, opendir, chdir)
- 只在开始和双重检查时运行 ps 一次
- 更快但可能有更多误报
- 在 32 位系统上比 unhide-linux quick reverse 快约 80 倍
- 在 64 位系统上快约 2 倍

**主要数据结构**：
```c
unsigned int proc_parent_pids[MAX_PID];
char *proc_tasks[MAX_PID];
char *ps_pids[MAX_PID];
char *messages_pids[MAX_PID];
```

## 核心函数分析 / Core Functions Analysis

### unhide-linux.c 核心函数

#### 1. get_max_pid()
```c
void get_max_pid(int* newmaxpid)
```
- 从 `/proc/sys/kernel/pid_max` 读取系统最大 PID
- 默认值：8388608 (MAX_PID)
- 用于确定需要扫描的 PID 范围

#### 2. checkps()
```c
int checkps(int tmppid, int checks)
```
- 核心验证函数
- 检查给定 PID 是否被 ps 命令看到
- 支持三种检查模式：
  - `PS_PROC` (0x01) - 检查进程
  - `PS_THREAD` (0x02) - 检查线程
  - `PS_MORE` (0x04) - 更多检查 (会话ID、进程组ID)
- 返回 1 如果找到，0 如果未找到

**实现要点**：
- 使用 popen() 执行 ps 命令
- 使用不同的 ps 命令格式：
  ```c
  #define COMMAND "ps --no-header -p %i o pid"      // 检查进程
  #define THREADS "ps --no-header -eL o lwp"        // 检查线程
  #define SESSION "ps --no-header -s %i o sess"     // 检查会话
  #define PGID "ps --no-header -eL o pgid"          // 检查进程组
  ```
- 解析输出并与期望的 PID 比较

#### 3. printbadpid()
```c
void printbadpid(int tmppid)
```
- 发现隐藏进程时的输出函数
- 设置全局标志 `found_HP = 1`
- 尝试从 /proc/[pid] 读取更多信息：
  - cmdline - 命令行参数
  - exe - 可执行文件链接
  - cwd - 当前工作目录
- 格式化输出隐藏进程的详细信息

### unhide-output.c 输出系统

#### 函数接口：

**msgln()** - 标准消息输出
```c
void msgln(FILE *unlog, int indent, const char* fmt, ...)
```
- 输出到 stdout
- 可选缩进
- 同时记录到日志文件

**warnln()** - 警告消息
```c
void warnln(int verbose, FILE *unlog, const char* fmt, ...)
```
- 输出到 stderr
- 受 verbose 级别控制
- 记录到日志文件

**die()** - 致命错误
```c
void die(FILE *unlog, const char* fmt, ...)
```
- 输出错误信息
- 退出程序，返回码 1

**init_log()** / **close_log()** - 日志管理
- 创建和关闭日志文件
- 默认文件名：unhide-linux.log
- 包含时间戳和使用的选项

## 命令行界面 / Command Line Interface

### unhide-linux 命令行选项

```
-V          显示版本并退出
-v          详细模式，显示警告信息 (可多次使用增加详细程度)
-h          显示帮助
-m          更多检查 (procfs, checkopendir, checkchdir 测试)
-r          在标准测试中使用 sysinfo 的替代版本
-f          写日志文件到当前目录 (unhide-linux.log)
-o          同 -f
-d          在 brute 测试中进行双重检查以避免误报
-H          输出更人性化的结果
```

### 标准测试 (Standard Tests)

```bash
unhide-linux brute       # 暴力破解所有 PID
unhide-linux proc        # /proc vs ps 对比
unhide-linux procall     # proc + procfs 组合
unhide-linux procfs      # procfs 遍历
unhide-linux quick       # 快速组合测试 (~20倍速)
unhide-linux reverse     # 反向验证 ps 输出
unhide-linux sys         # 系统调用扫描
```

### 基本测试 (Elementary Tests)

```bash
unhide-linux checkproc              # stat() /proc
unhide-linux checkchdir             # chdir() 测试
unhide-linux checkopendir           # opendir() 测试
unhide-linux checkreaddir           # readdir() 测试
unhide-linux checkgetprio           # getpriority() 测试
unhide-linux checkgetpgid           # getpgid() 测试
unhide-linux checkgetsid            # getsid() 测试
unhide-linux checkgetaffinity       # sched_getaffinity() 测试
unhide-linux checkgetparam          # sched_getparam() 测试
unhide-linux checkgetsched          # sched_getscheduler() 测试
unhide-linux checkRRgetinterval     # sched_rr_get_interval() 测试
unhide-linux checkkill              # kill() 测试
unhide-linux checknoprocps          # 纯系统调用对比
unhide-linux checksysinfo           # sysinfo() 测试
unhide-linux checksysinfo2          # sysinfo() 替代版本
unhide-linux checksysinfo3          # sysinfo() 第三版本
```

### unhide-tcp 命令行选项

```
-h          显示帮助
--brief     安静模式，不显示警告
-f          显示 fuser 输出 (Linux) 或 sockstat (FreeBSD)
-l          显示 lsof 输出
-n          使用 /bin/netstat 而非 /sbin/ss
-s          服务器快速扫描策略
-o          写日志文件
-V          显示版本
-v          详细模式
-H          人性化输出
```

## 图形界面 / GUI (unhideGui.py)

基于 Python 和 Tkinter 的图形界面：

### 主要功能：
1. **选项选择** - 复选框选择命令行选项
2. **测试选择** - 选择标准测试或基本测试
3. **TCP/UDP 扫描** - unhide-tcp 的图形界面
4. **实时输出** - 显示扫描结果
5. **工具提示** - 每个选项的详细说明

### 支持的测试：
- 所有标准测试 (brute, proc, procall, procfs, quick, reverse, sys)
- 所有基本测试
- unhide-tcp TCP/UDP 扫描

## 编译系统 / Build System

### 构建脚本：build_all.sh

```bash
gcc -Wall -O2 --static -pthread unhide-linux*.c unhide-output.c -o unhide-linux
gcc -Wall -O2 --static unhide_rb.c -o unhide_rb
gcc -Wall -O2 --static unhide-tcp.c unhide-tcp-fast.c unhide-output.c -o unhide-tcp
gcc -Wall -O2 --static unhide-posix.c -o unhide-posix
```

### 编译要点：

**关键编译标志**：
- `--static` - 静态链接，防止被篡改的系统库欺骗
- `-pthread` - 多线程支持 (unhide-linux)
- `-Wall -O2` - 警告和优化
- `-Wextra` - 额外警告 (README 中建议)

**特性定义**：
```c
#define _XOPEN_SOURCE 500  // 声明 getpgid() 等
#define _GNU_SOURCE        // 声明 sched_getaffinity()
```

### 依赖关系：

**构建依赖**：
- glibc-devel
- glibc-static-devel

**运行依赖**：
- unhide-tcp (Linux): iproute2, net-tools, lsof, psmisc
- unhide-tcp (FreeBSD): sockstat, lsof, netstat
- unhide-linux/posix/rb: procps

## 安全考虑 / Security Considerations

### 为什么静态链接？

**重要**：作为取证工具，unhide 构建为静态链接：
1. **防止库篡改** - 主机系统库可能被入侵
2. **避免 PRELINK 欺骗** - 预链接可能被利用
3. **独立性** - 不依赖可能被替换的系统库

### 权限要求

**必须以 root 运行**：
- 需要访问所有进程信息
- 需要读取 /proc 的所有内容
- 需要执行特权系统调用

### 误报问题

**可能的误报来源**：
1. **竞态条件** - 短生命周期进程
2. **PID 重用** - 快速创建和销毁的进程
3. **系统负载** - 高负载系统可能影响结果

**减少误报的方法**：
- 使用 `-d` 选项进行双重检查 (brute 测试)
- 在检查前后验证进程是否仍存在
- 使用多种检测方法交叉验证

## 测试套件 / Test Suite

### sanity.sh
- unhide-linux 的测试套件
- 验证各种检测方法
- 回归测试

### sanity-tcp.sh  
- unhide-tcp 的测试套件
- 测试 TCP/UDP 端口检测

## 代码质量特点 / Code Quality Features

### 良好的编码实践：

1. **模块化设计**
   - 功能分离到不同文件
   - 清晰的接口定义

2. **错误处理**
   - 检查系统调用返回值
   - errno 验证
   - 优雅降级

3. **资源管理**
   - 正确的 malloc/free
   - 文件描述符管理
   - 无内存泄漏设计

4. **可移植性**
   - 支持多个 Unix 变体
   - 条件编译 (#ifdef)
   - POSIX 兼容版本

5. **文档**
   - 多语言 README (英语、法语、西班牙语)
   - 详细的 man 页面
   - 代码注释

### 待改进领域 (根据 TODO)：

1. ☐ brute force 测试的可配置检查次数
2. ☐ 更多代码重构和优化
3. ☐ 邮件发送选项
4. ☐ 美化源代码 (更多注释、函数头)
5. ☐ 版本号系统
6. ☐ 国际化 (gettext)
7. ☐ 安装脚本或自动化工具 (autotools/cmake)
8. ✓ CVS/SVN/Git 仓库 (已完成)
9. ✓ 更多详细级别 (已完成)
10. ✓ 命令行解析优化 (已完成)

## 性能特征 / Performance Characteristics

### 速度比较：

**unhide-linux quick** vs **标准测试 (proc+procfs+sys)**
- 快约 20 倍
- 更多可能的误报

**unhide_rb** vs **unhide-linux quick reverse**
- 32 位系统: 快约 80 倍
- 64 位系统: 快约 2 倍
- 更多误报（更少测试）

### 扫描时间影响因素：

1. **maxpid** - 系统最大 PID 数量
2. **活跃进程数** - 当前运行的进程数
3. **检测方法** - 不同方法有不同的复杂度
4. **系统负载** - CPU 和 I/O 负载

### brute 测试时间：
- 需要创建和销毁大量进程/线程
- 在 maxpid=8388608 的系统上可能需要较长时间
- 双重检查 (-d) 会使时间翻倍

## 数据流 / Data Flow

### 典型检测流程：

```
1. 初始化
   ├─ 读取 maxpid
   ├─ 获取自身 PID (mypid)
   ├─ 解析命令行参数
   └─ 初始化日志文件

2. 收集信息
   ├─ 执行 ps 命令获取可见进程
   ├─ 扫描 /proc 文件系统
   └─ 调用系统调用

3. 对比分析
   ├─ 比较不同信息源
   ├─ 识别差异
   └─ 验证持续性 (避免竞态)

4. 报告结果
   ├─ 输出隐藏进程信息
   ├─ 记录到日志
   └─ 设置退出码

5. 清理
   ├─ 释放内存
   ├─ 关闭文件
   └─ 退出
```

## 关键算法 / Key Algorithms

### 1. 双重检查算法 (brute 测试)

```c
// 第一次扫描
for (i=301; i < maxpid; i++) {
    vpid = vfork();
    if (vpid == 0) _exit(0);
    if (errno == 0) {
        allpids[vpid] = 0;  // 标记为已占用
        waitpid(vpid, &status, 0);
    }
}

// 第二次扫描 (如果启用)
for (i=301; i < maxpid; i++) {
    vpid = vfork();
    if (vpid == 0) _exit(0);
    if (errno == 0) {
        allpids2[vpid] = 0;  // 标记为已占用
        waitpid(vpid, &status, 0);
    }
}

// 检查两次都未占用的 PID
for (y=0; y < maxpid; y++) {
    if ((allpids[y] != 0) && (allpids2[y] != 0)) {
        if (!checkps(allpids[y], PS_PROC | PS_THREAD | PS_MORE)) {
            printbadpid(allpids[y]);  // 发现隐藏进程
        }
    }
}
```

### 2. 快速组合算法 (quick 测试)

对每个 PID 执行多个轻量级测试：
```c
for (syspids = 1; syspids <= maxpid; syspids++) {
    found = 0;
    test_number = 0;
    
    // 执行 11 个测试
    if (kill(syspids, 0) == 0) { found++; test_number++; }
    if (stat(/proc/syspids) == 0) { found++; test_number++; }
    if (opendir(/proc/syspids)) { found++; test_number++; }
    if (chdir(/proc/syspids) == 0) { found++; test_number++; }
    // ... 更多系统调用测试
    
    // 如果找到但 ps 看不到
    if (found > 0 && !checkps(syspids)) {
        printbadpid(syspids);
    }
}
```

### 3. 反向验证算法 (reverse 测试)

```c
// 获取 ps 的所有输出
FILE *ps_output = popen("ps --no-header -eL o lwp,cmd", "r");

// 对每个 ps 报告的进程
while (fgets(line, ps_output)) {
    pid = parse_pid(line);
    
    // 验证在 /proc 中可见
    if (stat(/proc/pid) != 0) {
        report_issue("ps reports but not in /proc");
    }
    
    // 验证系统调用可访问
    if (kill(pid, 0) != 0) {
        report_issue("ps reports but kill fails");
    }
    
    // ... 更多验证
}
```

## 跨平台支持 / Cross-Platform Support

### Linux (>= 2.6)
- 完整功能
- unhide-linux (推荐)
- 所有检测技术

### 旧版 Linux (< 2.6)
- unhide-posix
- 有限功能
- 无 PID 暴力破解

### FreeBSD
- unhide-tcp 支持
- 使用 sockstat 替代 fuser
- 特殊的 netstat 命令

### OpenBSD
- unhide-tcp 基础支持
- 特定的 netstat 参数

### Solaris
- unhide-posix
- 基本功能

### 条件编译示例：

```c
#ifdef __linux__
    int use_ss = 1;
#else
    int use_ss = 0;
#endif

#if defined(__FreeBSD__) || defined(__FreeBSD_kernel__)
    char fuserTCPcommand[]= "sockstat -46 -p %d -P tcp";
#else
    char fuserTCPcommand[]= "fuser -v -n tcp %d 2>&1";
#endif
```

## 总结 / Summary

Unhide 是一个设计精良的取证工具，具有以下特点：

### 优势：
✓ 多种独立的检测技术
✓ 静态链接保证完整性
✓ 模块化、可维护的代码
✓ 跨平台支持
✓ 图形和命令行界面
✓ 详细的日志和报告
✓ 活跃维护 (最新版本 2024-05-09)

### 检测技术覆盖：
- /proc 文件系统 (4 种方法)
- 系统调用 (11 种调用)
- PID 空间暴力破解
- 组合和反向验证
- TCP/UDP 端口扫描

### 适用场景：
1. 系统安全审计
2. Rootkit 检测
3. 取证分析
4. 安全监控
5. 入侵检测

Unhide 通过多层次、多角度的检测方法，提供了一个强大的工具来发现系统中被恶意隐藏的进程和网络连接，是系统管理员和安全专家的重要工具。

---

**文档版本**: 1.0  
**分析日期**: 2024年12月12日  
**基于代码版本**: 20240509  
