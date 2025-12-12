# Source Code Analysis Documentation

## 文档概述 / Documentation Overview

本目录包含对 Unhide 项目的完整源代码分析文档。

This directory contains comprehensive source code analysis documentation for the Unhide project.

## 文档列表 / Document List

### 1. [SOURCE_CODE_ANALYSIS.md](./SOURCE_CODE_ANALYSIS.md)
**综合源代码分析 / Comprehensive Source Code Analysis**

这是一份双语（中文/英文）的详细分析文档，涵盖：

This is a bilingual (Chinese/English) detailed analysis covering:

- 项目概述和历史 / Project overview and history
- 源代码文件组织 / Source file organization
- 检测技术详解 / Detection techniques explained
- 核心函数分析 / Core function analysis
- 命令行界面文档 / CLI documentation
- 编译系统说明 / Build system explanation
- 安全考虑 / Security considerations
- 代码质量特点 / Code quality features

**适合读者 / Target Audience**: 
- 想要了解项目整体结构的开发者 / Developers wanting to understand overall structure
- 需要维护或扩展代码的贡献者 / Contributors who need to maintain or extend the code
- 对取证工具实现感兴趣的学习者 / Learners interested in forensic tool implementation

### 2. [ARCHITECTURE.md](./ARCHITECTURE.md)
**架构文档与图表 / Architecture Documentation with Diagrams**

这是一份英文的架构文档，包含：

This is an English architecture document containing:

- 系统架构图 / System architecture diagrams
- 检测技术架构 / Detection techniques architecture
- 数据流图 / Data flow diagrams
- 模块依赖关系 / Module dependencies
- 设计模式分析 / Design pattern analysis
- 性能考虑 / Performance considerations
- ASCII 艺术图表 / ASCII art diagrams

**适合读者 / Target Audience**:
- 系统架构师 / System architects
- 需要理解代码流程的开发者 / Developers needing to understand code flow
- 性能优化工程师 / Performance optimization engineers
- 技术审查人员 / Technical reviewers

## 快速导航 / Quick Navigation

### 想要了解... / Want to understand...

**整体项目结构？ / Overall project structure?**
→ 查看 SOURCE_CODE_ANALYSIS.md 的 "项目结构" 部分
→ See "Project Structure" in SOURCE_CODE_ANALYSIS.md

**检测技术如何工作？ / How detection techniques work?**
→ 查看 SOURCE_CODE_ANALYSIS.md 的 "架构设计" 部分
→ 查看 ARCHITECTURE.md 的 "Detection Techniques Architecture"
→ See "Architecture Design" in SOURCE_CODE_ANALYSIS.md
→ See "Detection Techniques Architecture" in ARCHITECTURE.md

**如何编译和使用？ / How to compile and use?**
→ 查看 SOURCE_CODE_ANALYSIS.md 的 "编译系统" 部分
→ 阅读主 README.txt
→ See "Build System" in SOURCE_CODE_ANALYSIS.md
→ Read main README.txt

**核心算法实现？ / Core algorithm implementation?**
→ 查看 SOURCE_CODE_ANALYSIS.md 的 "关键算法" 部分
→ 查看 ARCHITECTURE.md 的 "Key Design Patterns"
→ See "Key Algorithms" in SOURCE_CODE_ANALYSIS.md
→ See "Key Design Patterns" in ARCHITECTURE.md

**模块之间的关系？ / Module relationships?**
→ 查看 ARCHITECTURE.md 的 "Module Dependencies" 图
→ See "Module Dependencies" diagram in ARCHITECTURE.md

**性能特征？ / Performance characteristics?**
→ 查看 SOURCE_CODE_ANALYSIS.md 的 "性能特征" 部分
→ 查看 ARCHITECTURE.md 的 "Performance Considerations"
→ See "Performance Characteristics" in SOURCE_CODE_ANALYSIS.md
→ See "Performance Considerations" in ARCHITECTURE.md

## 关键发现 / Key Findings

### 优势 / Strengths

✓ **多层次检测** / Multi-layered Detection
- 6 种主要检测技术 / 6 main detection techniques
- 15+ 独立测试方法 / 15+ independent test methods
- 交叉验证机制 / Cross-validation mechanisms

✓ **安全设计** / Secure Design
- 静态链接防篡改 / Static linking prevents tampering
- 双重检查减少误报 / Double-check reduces false positives
- 竞态条件缓解 / Race condition mitigation

✓ **良好的代码组织** / Well-organized Code
- 模块化设计 / Modular design
- 清晰的接口 / Clear interfaces
- 适当的抽象层 / Proper abstraction layers

✓ **跨平台支持** / Cross-platform Support
- Linux (主要) / Linux (primary)
- FreeBSD, OpenBSD
- Solaris
- 通用 POSIX 系统 / Generic POSIX systems

### 架构亮点 / Architectural Highlights

**1. 策略模式 (Strategy Pattern)**
- 每种检测方法独立封装
- Each detection method independently encapsulated
- 易于添加新的检测技术
- Easy to add new detection techniques

**2. 门面模式 (Facade Pattern)**
- 统一的输出接口
- Unified output interface
- 简化日志和报告
- Simplified logging and reporting

**3. 模板方法模式 (Template Method)**
- 标准化的测试执行流程
- Standardized test execution flow
- 灵活的测试选择
- Flexible test selection

**4. 双重检查锁定 (Double-Checked Locking)**
- 减少暴力破解的误报
- Reduces false positives in brute force
- 性能与准确性的平衡
- Balance between performance and accuracy

## 代码指标 / Code Metrics

### 规模 / Size
```
总代码行数 / Total Lines:     ~5,000 lines of C code
主要模块 / Main Modules:      10 C files
头文件 / Headers:             3 header files
Python GUI:                   ~22KB (unhideGui.py)
```

### 复杂度 / Complexity
```
检测方法 / Detection Methods: 20+ functions
系统调用使用 / Syscalls Used:  15+ different syscalls
/proc 测试 / /proc Tests:      4 different approaches
组合测试 / Compound Tests:     2 meta tests
```

### 平台支持 / Platform Support
```
主要平台 / Primary:            Linux >= 2.6
次要平台 / Secondary:          FreeBSD, OpenBSD
遗留支持 / Legacy:             Linux < 2.6, Solaris
```

## 技术栈 / Technology Stack

### 核心技术 / Core Technologies
- **语言 / Language**: C (ANSI C with POSIX extensions)
- **构建 / Build**: GCC with manual Makefile
- **GUI**: Python 3 + Tkinter

### 依赖库 / Dependencies
- **glibc** (静态链接 / statically linked)
- **pthread** (多线程 / multithreading)
- **标准 POSIX API** / Standard POSIX APIs

### 外部工具 / External Tools
- ps (procps package)
- ss / netstat (networking tools)
- lsof (list open files)
- fuser / sockstat (port usage)

## 使用场景 / Use Cases

### 1. 系统安全审计 / System Security Audit
```bash
# 快速扫描 / Quick scan
sudo ./unhide-linux quick

# 全面扫描 / Comprehensive scan
sudo ./unhide-linux -vo procall sys
```

### 2. Rootkit 检测 / Rootkit Detection
```bash
# 暴力破解 PID 空间 / Brute force PID space
sudo ./unhide-linux -d brute

# 反向验证 / Reverse verification
sudo ./unhide-linux -v reverse
```

### 3. 网络端口审计 / Network Port Audit
```bash
# TCP/UDP 扫描 / TCP/UDP scan
sudo ./unhide-tcp -flov

# 服务器快速扫描 / Server quick scan
sudo ./unhide-tcp -s
```

### 4. 取证分析 / Forensic Analysis
```bash
# 详细日志记录 / Detailed logging
sudo ./unhide-linux -vvf brute proc sys

# 人性化输出 / Human-friendly output
sudo ./unhide-linux -H quick
```

## 学习路径 / Learning Path

### 初学者 / Beginners
1. 阅读主 README.txt 了解项目目的
   Read main README.txt to understand project purpose
2. 查看 SOURCE_CODE_ANALYSIS.md 的项目概述
   Check project overview in SOURCE_CODE_ANALYSIS.md
3. 尝试编译和运行基本测试
   Try compiling and running basic tests
4. 研究一个简单的检测方法（如 checkproc）
   Study a simple detection method (like checkproc)

### 中级开发者 / Intermediate Developers
1. 深入了解检测技术架构
   Deep dive into detection techniques architecture
2. 研究 checkps() 核心验证函数
   Study checkps() core validation function
3. 分析暴力破解算法
   Analyze brute force algorithm
4. 理解组合测试的实现
   Understand compound test implementation

### 高级开发者 / Advanced Developers
1. 研究所有 15+ 检测方法
   Study all 15+ detection methods
2. 分析性能优化技术
   Analyze performance optimization techniques
3. 理解安全设计决策
   Understand security design decisions
4. 考虑贡献新功能或优化
   Consider contributing new features or optimizations

## 贡献指南 / Contribution Guide

如果您想为 Unhide 项目做出贡献：

If you want to contribute to the Unhide project:

1. **阅读文档** / Read Documentation
   - 理解现有架构 / Understand existing architecture
   - 遵循代码风格 / Follow code style
   - 查看 TODO 文件了解需要的功能 / Check TODO for needed features

2. **测试** / Testing
   - 运行 sanity.sh 测试套件 / Run sanity.sh test suite
   - 在多个平台测试 / Test on multiple platforms
   - 验证无误报 / Verify no false positives

3. **文档** / Documentation
   - 更新相关文档 / Update relevant documentation
   - 添加代码注释 / Add code comments
   - 更新 man 页面 / Update man pages

## 参考资源 / References

### 项目资源 / Project Resources
- **官网 / Website**: http://www.unhide-forensics.info
- **源码 / Source**: http://sourceforge.net/projects/unhide/
- **许可 / License**: GPL v3

### 相关技术 / Related Technologies
- Linux /proc 文件系统 / Linux /proc filesystem
- POSIX 进程管理 / POSIX process management
- Linux 调度器 / Linux scheduler
- 网络编程 / Network programming

### 学习资源 / Learning Resources
- Linux man 页面 (proc, ps, kill, etc.)
- POSIX 标准文档 / POSIX standard documentation
- Linux 内核文档 / Linux kernel documentation
- 取证工具设计 / Forensic tool design

## 更新历史 / Update History

- **2024-12-12**: 创建初始分析文档 / Created initial analysis documents
  - SOURCE_CODE_ANALYSIS.md (双语版本 / Bilingual version)
  - ARCHITECTURE.md (英文版本 / English version)
  - ANALYSIS_README.md (本文件 / This file)

- **基于代码版本 / Based on Code Version**: 20240509 (2024-05-09)

## 许可证 / License

这些分析文档与 Unhide 项目使用相同的许可证：GPL v3

These analysis documents use the same license as the Unhide project: GPL v3

Copyright © 2024 Analysis Documentation
Copyright © 2010-2024 Yago Jesus & Patrick Gouin (Original Unhide code)

---

**文档维护者 / Document Maintainer**: GitHub Copilot Agent  
**最后更新 / Last Updated**: 2024-12-12  
**文档版本 / Document Version**: 1.0
