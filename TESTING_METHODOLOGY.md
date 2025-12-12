# Unhide 测试方法详解 / Testing Methodology

## 测试用例中如何隐藏进程 / How Processes Are Hidden in Test Cases

Unhide 的测试套件使用了一种巧妙的方法来模拟进程隐藏：**通过替换 `ps` 命令来伪造系统视图**。

The Unhide test suite uses a clever approach to simulate hidden processes: **replacing the `ps` command to fake the system view**.

## 核心测试原理 / Core Testing Principle

### PATH 劫持技术 / PATH Hijacking Technique

测试脚本在当前目录创建一个假的 `ps` 脚本，并通过修改 `PATH` 环境变量使其优先于真实的 `/bin/ps` 执行：

The test script creates a fake `ps` script in the current directory and modifies the `PATH` environment variable so it takes precedence over the real `/bin/ps`:

```bash
# 创建假的 ps 脚本 / Create fake ps script
cat <<EOF >./ps
#! /bin/bash
# 自定义行为 / Custom behavior
EOF

chmod 754 ./ps

# 将当前目录添加到 PATH 前面 / Add current directory to PATH
PATH=.:$PATH ./unhide-linux <test>
```

### 工作原理 / How It Works

```
unhide-linux 调用 ps
      ↓
查找 PATH 中的 ps 命令
      ↓
找到 ./ps (当前目录的假 ps)
      ↓
执行假 ps，返回修改后的输出
      ↓
unhide-linux 对比发现差异
      ↓
报告隐藏的进程
```

## 测试用例详解 / Test Cases Explained

### 测试 #1: 隐藏最后一个进程 / Test #1: Hide Last Process

**目的 / Purpose**: 模拟一个进程被隐藏

**方法 / Method**: 使用 `head -n-1` 删除 ps 输出的最后一行

```bash
cat <<EOF >./ps
#! /bin/bash
/bin/ps "\$@" | head -n-1
EOF
```

**预期结果 / Expected Result**: 
- 通常不会发现隐藏进程
- 因为最后一行往往是 ps 命令本身
- Unhide 会自动排除这个进程

**检测技术 / Detection Used**: `sys` (系统调用测试)

---

### 测试 #2: 隐藏所有进程 / Test #2: Hide All Processes

**目的 / Purpose**: 模拟 ps 命令完全失败

**方法 / Method**: ps 脚本直接返回失败

```bash
cat <<EOF >./ps
#! /bin/bash
false
EOF
```

**预期结果 / Expected Result**: 
- 所有进程都会被报告为隐藏
- 因为 ps 没有返回任何进程

**检测技术 / Detection Used**: `procall` (proc + procfs 组合)

---

### 测试 #3: 添加假进程 (sysinfo) / Test #3: Add Fake Process (sysinfo)

**目的 / Purpose**: 测试 sysinfo 系统调用检测

**方法 / Method**: 在 ps 输出中添加一个不存在的进程

```bash
cat <<EOF >./ps
#! /bin/bash
/bin/ps "\$@"
echo 65535  my_false_proc
EOF
```

**工作原理 / How It Works**:
1. `sysinfo()` 系统调用报告 N 个进程
2. ps 显示 N+1 个进程 (包含假进程)
3. Unhide 检测到数量不匹配

**预期结果 / Expected Result**:
```
"1 HIDDEN Process Found sysinfo.procs reports xxx processes and ps sees xxx-1 processes"
```

**检测技术 / Detection Used**: `checksysinfo`, `checksysinfo2`

**注意 / Note**: 由于现代系统进程数量大，测试可能不可靠

---

### 测试 #4: 反向检测假进程 / Test #4: Reverse Detection of Fake Process

**目的 / Purpose**: 测试反向验证功能

**方法 / Method**: 同样添加假进程，但使用 reverse 测试

```bash
cat <<EOF >./ps
#! /bin/bash
/bin/ps "\$@"
echo 65535  my_false_proc
EOF
```

**工作原理 / How It Works**:
1. ps 报告进程 65535 (my_false_proc)
2. Unhide 尝试在 /proc/65535 中验证
3. /proc/65535 不存在
4. 检测到伪造的进程

**预期结果 / Expected Result**: 
- 显示 "my_false_proc" 作为伪造进程

**检测技术 / Detection Used**: `reverse` (反向验证)

---

### 测试 #5: 长命令行假进程 / Test #5: Fake Process with Long Command Line

**目的 / Purpose**: 测试处理超长命令行 (>1023 字符)

**方法 / Method**: 添加一个命令行长度超过 1023 字符的假进程

```bash
cat <<EOF >./ps
#! /bin/bash
/bin/ps "\$@"
echo 65535  my_false_procmy_false_procmy_false_proc... (重复多次)
EOF
```

**测试目标 / Test Target**:
- 缓冲区溢出保护
- 长字符串处理
- 边界条件

**检测技术 / Detection Used**: `reverse`

---

### 测试 #6: 非数字 PID / Test #6: Non-numeric PID

**目的 / Purpose**: 测试错误输入处理

**方法 / Method**: 添加一个 PID 为非数字的假进程

```bash
cat <<EOF >./ps
#! /bin/bash
/bin/ps "\$@"
echo abcde bad_proc_number
EOF
```

**预期结果 / Expected Result**:
```
"Warning : No numeric pid found on ps output line, skip line"
```

**测试目标 / Test Target**:
- 输入验证
- 错误处理
- 不崩溃

**检测技术 / Detection Used**: `reverse` (详细模式)

---

### 测试 #7: 临时进程 / Test #7: Temporary Process

**目的 / Purpose**: 测试短生命周期进程

**方法 / Method**: 使用环境变量控制，只在第一次调用时添加假进程

```bash
cat <<'EOF' >./ps
#! /bin/bash
/bin/ps "$@"
if [[ -z "${UNH_PASSAGE}" ]]; then
   export UNH_PASSAGE=1
   echo abcde bad_proc_number
fi
EOF
```

**注意 / Note**: 此测试目前不工作（注释说明）

**检测技术 / Detection Used**: `quick`

---

## unhide-tcp 测试方法 / unhide-tcp Testing Method

### 测试 #0: 隐藏所有端口 / Test #0: Hide All Ports

**方法 / Method**: 替换 `ss`/`netstat` 命令使其失败

```bash
cat <<EOF >./ss
#!/bin/sh
false
EOF

PATH=.:$PATH ./unhide-tcp -fl
```

**预期结果 / Expected Result**: 所有监听端口都被报告为隐藏

---

### 测试 #1: 隐藏特定端口 / Test #1: Hide Specific Port

**目的 / Purpose**: 模拟 CUPS 端口 (631) 被隐藏

**方法 / Method**: 过滤掉端口 631 的输出

```bash
# Linux 版本
if [ "$4" != ":631" ]; then
    /sbin/ss $@
else
    echo "Le 4ème paramètre est : $4" >&2
fi

# FreeBSD 版本
/usr/bin/netstat $@ | grep -v 631
```

**工作原理 / How It Works**:
1. 真实的 ss/netstat 不显示端口 631
2. Unhide 尝试绑定端口 631
3. 绑定失败（因为 CUPS 正在使用）
4. 检测到隐藏的端口

**预期结果 / Expected Result**: 报告端口 631 为隐藏端口

---

## 测试设计的巧妙之处 / Clever Test Design

### 1. 无需 Root 权限模拟 Rootkit
- 不需要实际安装 rootkit
- 不需要修改内核
- 不需要 root 权限
- 通过 PATH 环境变量即可模拟

### 2. 可重复和安全
- 测试完全在用户空间
- 不影响系统安全
- 可以随时运行
- 自动清理 (删除假的 ps/ss)

### 3. 覆盖多种场景
- ✓ 隐藏进程
- ✓ 伪造进程
- ✓ 错误处理
- ✓ 边界条件
- ✓ 短生命周期进程

### 4. 跨平台兼容
- Linux: 使用 ss
- FreeBSD: 使用 netstat
- 条件编译处理差异

## 测试的局限性 / Test Limitations

### 1. 不测试真实 Rootkit
- 只测试 Unhide 的检测逻辑
- 不测试对抗真实恶意软件
- 不测试内核级隐藏

### 2. 竞态条件难以测试
- 短生命周期进程测试不完善
- 测试 #7 明确标注为不工作

### 3. 依赖系统环境
- 需要真实的 /bin/ps
- 需要真实的 /proc 文件系统
- 可能受系统负载影响

## 如何运行测试 / How to Run Tests

### 运行所有测试 / Run All Tests

```bash
# 进程检测测试 / Process detection tests
sudo ./sanity.sh

# TCP/UDP 端口检测测试 / Port detection tests
sudo ./sanity-tcp.sh
```

### 运行单个测试 / Run Individual Test

可以手动执行测试脚本中的某个部分：

```bash
# 创建假 ps / Create fake ps
cat <<EOF >./ps
#! /bin/bash
/bin/ps "\$@" | head -n-1
EOF
chmod 754 ./ps

# 运行测试 / Run test
PATH=.:$PATH ./unhide-linux sys

# 清理 / Cleanup
rm -f ./ps
```

## 测试输出示例 / Test Output Example

### 成功检测示例 / Successful Detection Example

```
Test #2
Don't call ps : let all processes appear hidden.
This should find all processes as hidden..

[*]Searching for Hidden processes through comparison...
Found HIDDEN PID: 1
Found HIDDEN PID: 2
Found HIDDEN PID: 3
...
```

### 反向检测示例 / Reverse Detection Example

```
Test #4
Call ps, but add a faked process.
This should show "my_false_proc" as faked process

[*]Reverse search, validating processes from ps...
Warning: Process 65535 seen by ps but not found in /proc
Found FAKED PID: 65535 (my_false_proc)
```

## 总结 / Summary

Unhide 的测试方法体现了优秀的测试设计：

1. **创造性模拟** / Creative Simulation
   - 通过 PATH 劫持模拟 rootkit 行为
   - 无需实际的恶意软件

2. **全面覆盖** / Comprehensive Coverage
   - 测试隐藏、伪造、错误处理
   - 覆盖多种检测技术

3. **安全可靠** / Safe and Reliable
   - 不影响系统
   - 可重复执行
   - 自动清理

4. **实用价值** / Practical Value
   - 验证检测逻辑正确性
   - 回归测试
   - 开发者信心保证

这种测试方法是**取证工具开发的最佳实践**，值得其他安全工具借鉴。

---

**文档版本 / Version**: 1.0  
**创建日期 / Created**: 2024-12-12  
**基于代码版本 / Based on**: sanity.sh & sanity-tcp.sh from Unhide 20240509
