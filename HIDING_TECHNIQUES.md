# 常见进程隐藏技术 / Common Process Hiding Techniques

## 概述 / Overview

本文档介绍 rootkit 和恶意软件常用的进程隐藏技术，以及 Unhide 如何检测这些隐藏方法。

This document describes common process hiding techniques used by rootkits and malware, and how Unhide detects them.

---

## 一、用户态隐藏技术 / User-Space Hiding Techniques

### 1. 库函数劫持 / Library Function Hijacking

#### 1.1 LD_PRELOAD 劫持

**技术原理 / Technique**:
- 通过 `LD_PRELOAD` 环境变量注入恶意共享库
- 拦截 `readdir()`, `opendir()` 等函数
- 过滤掉特定进程的返回结果

**示例代码 / Example**:
```c
// 恶意库拦截 readdir
struct dirent *readdir(DIR *dirp) {
    struct dirent *dir = original_readdir(dirp);
    if (dir && strcmp(dir->d_name, "evil_pid") == 0) {
        // 跳过恶意进程
        return readdir(dirp);
    }
    return dir;
}
```

**Unhide 检测方法 / Detection**:
- ✅ **静态链接**: Unhide 使用 `--static` 编译，不依赖系统库
- ✅ **直接系统调用**: 绕过被劫持的库函数
- ✅ **多重验证**: 使用 `/proc`, 系统调用, ps 命令交叉验证

**相关检测 / Related Tests**:
- `checkreaddir()` - 直接系统调用验证
- `checkopendir()` - opendir 验证
- `checkchdir()` - chdir 验证

---

#### 1.2 符号链接劫持 / Symbol Hijacking

**技术原理 / Technique**:
- 替换系统库中的符号
- 重定向函数调用到恶意代码

**Unhide 检测方法 / Detection**:
- ✅ **静态链接**: 不加载动态库，避免符号污染

---

### 2. /proc 文件系统操纵 / /proc Filesystem Manipulation

#### 2.1 进程名伪装 / Process Name Spoofing

**技术原理 / Technique**:
- 修改 `/proc/[pid]/cmdline`
- 修改 `/proc/[pid]/comm`
- 修改进程的 `argv[0]`

**示例代码 / Example**:
```c
// 修改进程名
strcpy(argv[0], "legitimate_name");
prctl(PR_SET_NAME, "legitimate_name");
```

**Unhide 检测方法 / Detection**:
- ✅ **多重检查**: 同时检查 cmdline, exe, cwd
- ✅ **PID 验证**: 基于 PID 而非进程名检测
- ⚠️ **局限**: 无法检测单纯的进程名伪装（不是真正的隐藏）

**相关检测 / Related Tests**:
- `printbadpid()` - 显示 cmdline, exe, cwd 完整信息

---

#### 2.2 /proc 条目删除 / /proc Entry Removal

**技术原理 / Technique**:
- 通过内核模块删除 `/proc/[pid]` 目录
- 使用户态工具看不到进程

**Unhide 检测方法 / Detection**:
- ✅ **系统调用验证**: 使用 `getpriority()`, `kill()` 等系统调用
- ✅ **暴力破解**: 尝试所有可能的 PID
- ✅ **内核信息**: 使用 `sysinfo()` 获取真实进程数

**相关检测 / Related Tests**:
- `checkgetpriority()` - getpriority 系统调用
- `checkgetpgid()` - getpgid 系统调用
- `checkgetsid()` - getsid 系统调用
- `checkkill()` - kill(pid, 0) 测试
- `brute()` - 暴力破解 PID 空间

---

### 3. ps 命令劫持 / ps Command Hijacking

#### 3.1 PATH 环境变量劫持

**技术原理 / Technique**:
- 将恶意 `ps` 放在 PATH 前面
- 过滤掉特定进程

**示例代码 / Example**:
```bash
#!/bin/bash
# 恶意 ps
/bin/ps.real "$@" | grep -v evil_process
```

**Unhide 检测方法 / Detection**:
- ✅ **绝对路径**: 可以指定 ps 的完整路径
- ✅ **不依赖 ps**: 主要依靠系统调用和 /proc
- ⚠️ **假设**: ps 命令可能被污染，因此使用多种方法交叉验证

**相关检测 / Related Tests**:
- `checkps()` - 验证 ps 输出
- `checkallnoprocps()` - 完全不依赖 ps 的检测

---

#### 3.2 ps 二进制替换

**技术原理 / Technique**:
- 直接替换 `/bin/ps` 二进制文件
- 内置过滤逻辑

**Unhide 检测方法 / Detection**:
- ✅ **多重信息源**: 不完全依赖 ps
- ✅ **反向验证**: `reverse` 测试验证 ps 输出的真实性

**相关检测 / Related Tests**:
- `checkallreverse()` - 反向验证 ps 输出

---

## 二、内核态隐藏技术 / Kernel-Space Hiding Techniques

### 1. 系统调用表劫持 / System Call Table Hijacking

#### 1.1 sys_call_table 修改

**技术原理 / Technique**:
- 修改内核系统调用表
- 拦截 `getdents()`, `getdents64()` 等系统调用
- 过滤目录列表中的恶意进程

**示例伪代码 / Pseudo Code**:
```c
// 内核模块劫持 getdents
asmlinkage long hook_getdents(unsigned int fd, 
                               struct linux_dirent *dirp, 
                               unsigned int count) {
    long ret = original_getdents(fd, dirp, count);
    // 过滤掉恶意进程 PID
    filter_process_entries(dirp, count);
    return ret;
}
```

**Unhide 检测方法 / Detection**:
- ✅ **多种系统调用**: 使用多个不同的系统调用
- ✅ **直接调用**: 绕过可能被劫持的高层接口
- ✅ **交叉验证**: 对比不同系统调用的结果

**相关检测 / Related Tests**:
- 所有 `check*()` 函数使用不同的系统调用路径

---

#### 1.2 VFS 层劫持 / VFS Layer Hijacking

**技术原理 / Technique**:
- 劫持 VFS (Virtual File System) 操作
- 修改 `iterate_dir()`, `filldir()` 等函数
- 在文件系统层面隐藏进程

**Unhide 检测方法 / Detection**:
- ✅ **绕过 VFS**: 使用不依赖 VFS 的系统调用
- ✅ **PID 暴力破解**: 直接测试 PID 存在性

**相关检测 / Related Tests**:
- `brute()` - 完全绕过文件系统

---

### 2. 进程链表操纵 / Process List Manipulation

#### 2.1 任务队列隐藏 / Task Queue Hiding

**技术原理 / Technique**:
- 从内核任务队列 (`task_struct` 链表) 中移除进程
- 进程仍在运行，但对大部分工具不可见

**示例伪代码 / Pseudo Code**:
```c
// 从任务队列中移除
list_del(&task->tasks);
list_del(&task->sibling);
```

**Unhide 检测方法 / Detection**:
- ✅ **PID 暴力破解**: 尝试分配所有 PID
- ✅ **系统信息对比**: `sysinfo()` 报告的进程数 vs 可见进程数
- ⚠️ **部分有效**: 完全从调度器移除的进程可能检测不到

**相关检测 / Related Tests**:
- `brute()` - 暴力破解检测
- `checksysinfo()` - 进程数量对比

---

#### 2.2 PID 命名空间操纵 / PID Namespace Manipulation

**技术原理 / Technique**:
- 使用 Linux PID 命名空间特性
- 在不同命名空间中隐藏进程

**Unhide 检测方法 / Detection**:
- ⚠️ **局限**: 难以检测不同命名空间的进程
- ✅ **容器感知**: 需要在容器内部运行 Unhide

---

### 3. 直接内存操作 / Direct Kernel Object Manipulation (DKOM)

#### 3.1 进程对象隐藏

**技术原理 / Technique**:
- 直接修改内核数据结构
- 隐藏 `task_struct` 对象
- 移除进程相关的所有引用

**Unhide 检测方法 / Detection**:
- ✅ **资源检测**: 检测系统资源使用异常
- ⚠️ **高级隐藏**: 完全从内核移除的进程极难检测

---

## 三、网络端口隐藏技术 / Network Port Hiding Techniques

### 1. netstat/ss 劫持 / netstat/ss Hijacking

**技术原理 / Technique**:
- 替换或劫持 `netstat`, `ss` 命令
- 过滤掉恶意端口

**Unhide-TCP 检测方法 / Detection**:
- ✅ **端口扫描**: 暴力扫描所有端口 (1-65535)
- ✅ **连接尝试**: 尝试连接所有端口
- ✅ **对比验证**: 扫描结果 vs ss/netstat 输出

**相关测试 / Related Tests**:
- `unhide-tcp` 完整端口扫描

---

### 2. /proc/net/ 文件操纵 / /proc/net/ File Manipulation

**技术原理 / Technique**:
- 劫持 `/proc/net/tcp`, `/proc/net/udp` 的读取
- 过滤掉特定端口信息

**Unhide-TCP 检测方法 / Detection**:
- ✅ **直接套接字操作**: 尝试绑定端口
- ✅ **不依赖 /proc/net**: 通过实际连接测试

---

### 3. 内核网络栈劫持 / Kernel Network Stack Hijacking

**技术原理 / Technique**:
- 劫持内核网络栈函数
- 隐藏套接字信息

**Unhide-TCP 检测方法 / Detection**:
- ✅ **端口绑定测试**: 实际尝试使用端口
- ✅ **连接测试**: 主动连接测试

---

## 四、高级隐藏技术 / Advanced Hiding Techniques

### 1. Hypervisor 层隐藏 / Hypervisor-Level Hiding

**技术原理 / Technique**:
- 在虚拟化层隐藏进程
- 拦截虚拟机的系统调用
- Blue Pill, SubVirt 等技术

**Unhide 检测方法 / Detection**:
- ❌ **无法检测**: Unhide 运行在 guest OS，无法检测 hypervisor 层
- ⚠️ **需要特殊工具**: 需要从 host OS 或专门的检测工具

---

### 2. 硬件层隐藏 / Hardware-Level Hiding

**技术原理 / Technique**:
- SMM (System Management Mode) rootkit
- BIOS/UEFI 固件级别的隐藏
- Intel ME (Management Engine) 利用

**Unhide 检测方法 / Detection**:
- ❌ **无法检测**: 运行在更底层，OS 无法检测
- ⚠️ **需要固件工具**: 需要专门的固件分析工具

---

### 3. 反调试和反检测技术 / Anti-Debugging & Anti-Detection

#### 3.1 检测分析工具

**技术原理 / Technique**:
- 检测是否有分析工具在运行
- 发现 Unhide 等工具时隐藏或停止

**Unhide 检测方法 / Detection**:
- ⚠️ **难以对抗**: 名称可以被检测
- ✅ **快速扫描**: 尽快完成检测

---

#### 3.2 时间依赖的隐藏

**技术原理 / Technique**:
- 只在特定时间出现
- 短生命周期进程

**Unhide 检测方法 / Detection**:
- ✅ **双重检查**: `-d` 选项进行二次验证
- ⚠️ **竞态条件**: 可能产生误报

**相关测试 / Related Tests**:
- `brute()` 双重检查模式

---

## 五、Unhide 检测策略总结 / Unhide Detection Strategy Summary

### 多层次防御 / Multi-Layer Defense

```
┌─────────────────────────────────────────────────────────┐
│ 隐藏技术层次                    Unhide 对策              │
├─────────────────────────────────────────────────────────┤
│ 用户态库劫持     →  静态链接 + 直接系统调用             │
│ /proc 操纵       →  系统调用验证 + 暴力破解             │
│ ps 劫持          →  多重信息源 + 反向验证               │
│ 系统调用劫持     →  多种系统调用交叉验证               │
│ 进程链表操纵     →  PID 暴力破解 + sysinfo 对比        │
│ 网络栈劫持       →  端口扫描 + 连接测试                │
│ 高级技术         →  部分无法检测，需要其他工具          │
└─────────────────────────────────────────────────────────┘
```

### 检测技术矩阵 / Detection Technique Matrix

| 隐藏技术 | 检测方法 | 有效性 | 局限性 |
|---------|---------|--------|--------|
| LD_PRELOAD | 静态链接 | ✅ 高 | 无 |
| /proc 删除 | 系统调用 | ✅ 高 | 需 root |
| ps 劫持 | 交叉验证 | ✅ 高 | 依赖 ps |
| 系统调用表劫持 | 多种系统调用 | ✅ 中 | 高级劫持可绕过 |
| 任务队列隐藏 | PID 暴力破解 | ✅ 中 | 完全移除难检测 |
| DKOM | 资源异常检测 | ⚠️ 低 | 高级技术难检测 |
| Hypervisor | N/A | ❌ 无 | 需要 host 访问 |
| 硬件层 | N/A | ❌ 无 | 需要固件工具 |

---

## 六、实战案例 / Real-World Examples

### 案例 1: Adore-ng Rootkit

**隐藏技术**:
- 劫持 `getdents64()` 系统调用
- 从 `/proc` 中隐藏特定 PID

**Unhide 检测**:
```bash
sudo ./unhide-linux sys
# 使用系统调用检测，绕过被劫持的 getdents
```

**结果**: ✅ 可检测

---

### 案例 2: Diamorphine Rootkit

**隐藏技术**:
- LKM (可加载内核模块)
- 从任务队列移除进程
- 劫持 `/proc` 读取

**Unhide 检测**:
```bash
sudo ./unhide-linux -d brute
# 双重检查的暴力破解
```

**结果**: ✅ 可检测

---

### 案例 3: Reptile Rootkit

**隐藏技术**:
- 多种技术组合
- 文件隐藏 + 进程隐藏 + 端口隐藏

**Unhide 检测**:
```bash
sudo ./unhide-linux quick
sudo ./unhide-tcp -flov
# 组合使用进程和端口检测
```

**结果**: ✅ 大部分可检测

---

## 七、防御建议 / Defense Recommendations

### 1. 预防性措施 / Preventive Measures

- 🔒 **安全启动**: 启用 Secure Boot
- 🔒 **内核完整性**: 使用 IMA/EVM
- 🔒 **最小权限**: 限制 root 访问
- 🔒 **SELinux/AppArmor**: 强制访问控制

### 2. 检测最佳实践 / Detection Best Practices

- 📊 **定期扫描**: 使用 cron 定期运行 Unhide
- 📊 **多工具组合**: 不要只依赖单一工具
- 📊 **基线对比**: 建立正常系统的基线
- 📊 **日志分析**: 记录和分析 Unhide 输出

### 3. 响应策略 / Response Strategy

- 🚨 **隔离系统**: 发现隐藏进程立即隔离
- 🚨 **取证分析**: 保存内存快照和磁盘镜像
- 🚨 **根因分析**: 找出入侵途径
- 🚨 **重建系统**: 从可信介质重建

---

## 八、检测局限性 / Detection Limitations

### Unhide 无法检测的情况 / What Unhide Cannot Detect

1. **完全从调度器移除的进程**
   - 不占用 PID
   - 不响应任何系统调用
   - 只能通过内存取证发现

2. **Hypervisor/硬件层隐藏**
   - 运行在比 OS 更底层
   - 需要专门的检测工具

3. **特权容器/命名空间**
   - 不同命名空间的进程
   - 需要在每个命名空间运行

4. **高级 DKOM 技术**
   - 完全操纵内核对象
   - 绕过所有检测点

### 误报情况 / False Positives

- ⚠️ **短生命周期进程**: 快速创建和销毁的进程
- ⚠️ **系统负载高**: 竞态条件增加
- ⚠️ **容器环境**: 命名空间隔离可能导致误判

---

## 九、延伸阅读 / Further Reading

### 相关工具 / Related Tools

- **rkhunter**: Rootkit Hunter
- **chkrootkit**: Check Rootkit
- **AIDE**: Advanced Intrusion Detection Environment
- **Volatility**: 内存取证工具

### 技术文档 / Technical Documentation

- Linux 内核文档: `/proc` 文件系统
- LKM 编程指南
- Rootkit 分析技术
- 内存取证方法

### Unhide 项目文档 / Unhide Documentation

- `SOURCE_CODE_ANALYSIS.md` - 源代码分析
- `ARCHITECTURE.md` - 架构设计
- `TESTING_METHODOLOGY.md` - 测试方法
- `README.txt` - 使用说明

---

## 总结 / Summary

进程隐藏技术层出不穷，从简单的库劫持到复杂的内核操纵，再到硬件层的隐藏。Unhide 通过以下策略提供多层次的检测能力：

Process hiding techniques range from simple library hijacking to complex kernel manipulation and hardware-level concealment. Unhide provides multi-layered detection through:

1. **静态链接**: 避免用户态劫持
2. **多重验证**: 使用多种系统调用和信息源
3. **暴力破解**: 覆盖完整的 PID 空间
4. **交叉验证**: 对比不同方法的结果
5. **反向检查**: 验证工具自身的输出

虽然无法检测所有高级隐藏技术，但 Unhide 对常见的 rootkit 和恶意软件隐藏技术有很好的检测效果。

While it cannot detect all advanced hiding techniques, Unhide is effective against common rootkit and malware hiding methods.

**关键原则 / Key Principle**: 
> "没有单一工具能检测所有威胁。防御需要多层次、多工具的组合策略。"
> 
> "No single tool can detect all threats. Defense requires a multi-layered, multi-tool strategy."

---

**文档版本 / Version**: 1.0  
**创建日期 / Created**: 2024-12-12  
**相关项目 / Related**: Unhide 20240509
