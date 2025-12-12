# Unhide Architecture Overview

## System Architecture Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                        Unhide Toolkit                          │
└────────────────────────────────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
┌───────▼────────┐   ┌────────▼─────────┐   ┌───────▼────────┐
│ unhide-linux   │   │  unhide-tcp      │   │  unhide_rb     │
│   (Main Tool)  │   │  (Port Scanner)  │   │  (Lightweight) │
└───────┬────────┘   └────────┬─────────┘   └───────┬────────┘
        │                     │                      │
        │                     │                      │
┌───────▼──────────────────────────────────────────────────────┐
│                    unhide-output.c                           │
│              (Common Output & Logging Layer)                 │
└──────────────────────────────────────────────────────────────┘
```

## Detection Techniques Architecture

```
unhide-linux Detection Methods:
═══════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────┐
│                    1. /proc Filesystem Tests                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  checkproc  │  │  checkchdir  │  │ checkopendir │       │
│  │   (stat)    │  │   (chdir)    │  │  (opendir)   │       │
│  └─────────────┘  └──────────────┘  └──────────────┘       │
│  ┌─────────────┐                                            │
│  │checkreaddir │                                            │
│  │  (readdir)  │                                            │
│  └─────────────┘                                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                 2. System Call Tests (11 types)              │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ getpriority  │  │   getpgid    │  │   getsid     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │sched_getaff  │  │sched_getparam│  │sched_getsched│      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │sched_rr_int  │  │     kill     │                        │
│  └──────────────┘  └──────────────┘                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  sysinfo     │  │  sysinfo2    │  │  sysinfo3    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                 3. Brute Force PID Scan                      │
├─────────────────────────────────────────────────────────────┤
│  Method 1: vfork() all PIDs (1 to maxpid)                   │
│  Method 2: pthread_create() all TIDs                        │
│                                                              │
│  With -d: Double check to reduce false positives            │
│  ┌────────┐      ┌────────┐      ┌──────────┐              │
│  │ Scan 1 │─────▶│ Scan 2 │─────▶│ Compare  │              │
│  └────────┘      └────────┘      └──────────┘              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              4. Compound Tests (Meta Tests)                  │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────┐                     │
│  │   quick: Fast combined test        │                     │
│  │   - proc + procfs + sys            │                     │
│  │   - ~20x faster                    │                     │
│  │   - More false positives           │                     │
│  └────────────────────────────────────┘                     │
│  ┌────────────────────────────────────┐                     │
│  │   reverse: Verify ps output        │                     │
│  │   - Check ps against /proc & sys   │                     │
│  │   - Detect fake processes          │                     │
│  └────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
```

## Data Flow Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Initialization                          │
│  - Parse command line arguments                             │
│  - Read maxpid from /proc/sys/kernel/pid_max               │
│  - Get own PID (mypid)                                      │
│  - Initialize log file (if -f/-o specified)                │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│                   Information Collection                     │
│                                                              │
│  ┌────────────────┐    ┌─────────────────┐                 │
│  │  Execute ps    │    │  Scan /proc     │                 │
│  │  commands      │    │  filesystem     │                 │
│  └────────┬───────┘    └────────┬────────┘                 │
│           │                     │                           │
│           └──────────┬──────────┘                           │
│                      │                                      │
│           ┌──────────▼──────────┐                          │
│           │ System Call Probing │                          │
│           │  (11 different)     │                          │
│           └─────────────────────┘                          │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│                   Comparison & Analysis                      │
│                                                              │
│  For each PID (1 to maxpid):                                │
│  ┌──────────────────────────────────────────────┐          │
│  │ 1. Collect info from multiple sources        │          │
│  │ 2. Check if visible to ps (checkps)          │          │
│  │ 3. Verify persistence (avoid race conditions)│          │
│  │ 4. If mismatch found:                         │          │
│  │    ├─ Found by /proc but not ps? → Hidden!   │          │
│  │    ├─ Found by syscall but not ps? → Hidden! │          │
│  │    └─ Found by brute but not ps? → Hidden!   │          │
│  └──────────────────────────────────────────────┘          │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│                     Result Reporting                         │
│                                                              │
│  For each hidden process found:                             │
│  ┌──────────────────────────────────────────────┐          │
│  │ printbadpid(pid):                             │          │
│  │  - Set found_HP = 1                           │          │
│  │  - Read /proc/[pid]/cmdline                   │          │
│  │  - Read /proc/[pid]/exe link                  │          │
│  │  - Read /proc/[pid]/cwd link                  │          │
│  │  - Output to stdout & log file                │          │
│  └──────────────────────────────────────────────┘          │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│                        Cleanup                               │
│  - Free allocated memory                                    │
│  - Close log file                                           │
│  - Exit with code: 0 (none found) or 1 (found hidden)      │
└─────────────────────────────────────────────────────────────┘
```

## Module Dependencies

```
┌──────────────────────────────────────────────────────────────┐
│                      unhide-linux.c                          │
│  - Main entry point                                          │
│  - Command line parsing (parse_args)                         │
│  - Core validation (checkps)                                 │
│  - Output formatting (printbadpid)                           │
│  - Configuration (get_max_pid)                               │
└────┬──────────────┬──────────────┬─────────────┬────────────┘
     │              │              │             │
     ▼              ▼              ▼             ▼
┌─────────┐  ┌────────────┐  ┌──────────┐  ┌──────────────┐
│bruteforce│  │  procfs    │  │ syscall  │  │  compound    │
│   .c     │  │    .c      │  │   .c     │  │     .c       │
├─────────┤  ├────────────┤  ├──────────┤  ├──────────────┤
│brute()  │  │checkproc() │  │getprio() │  │checkallquick()│
│         │  │checkchdir()│  │getpgid() │  │checkallrev() │
└────┬────┘  │checkopendir│  │getsid()  │  └──────────────┘
     │       │checkreaddir│  │kill()    │
     │       └────────────┘  │sysinfo() │
     │                       └──────────┘
     │
     └───────────────┬──────────────────────────────┐
                     │                              │
                     ▼                              ▼
            ┌──────────────────┐          ┌──────────────┐
            │ unhide-output.c  │          │pthread lib   │
            ├──────────────────┤          ├──────────────┤
            │msgln()           │          │pthread_create│
            │warnln()          │          │pthread_join  │
            │die()             │          └──────────────┘
            │init_log()        │
            │close_log()       │
            └──────────────────┘
```

## unhide-tcp Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      unhide-tcp.c                            │
│                  (Port Detection Main)                       │
└─────────────┬───────────────────────────────────────────────┘
              │
              ├────────────────────┬─────────────────────────┐
              │                    │                         │
              ▼                    ▼                         ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Method 1:       │  │  Method 2:       │  │  Verification:   │
│  Brute Force     │  │  Probe Unused    │  │  External Tools  │
├──────────────────┤  ├──────────────────┤  ├──────────────────┤
│ Try to bind all  │  │ Get ss/netstat   │  │ fuser (Linux)    │
│ TCP ports 1-65535│  │ output           │  │ sockstat(FreeBSD)│
│                  │  │                  │  │                  │
│ Try to bind all  │  │ Find "unused"    │  │ lsof             │
│ UDP ports 1-65535│  │ ports            │  │                  │
│                  │  │                  │  │ netstat          │
│ Compare with     │  │ Try to connect   │  │                  │
│ ss/netstat       │  │ to them          │  │ ss               │
└──────────────────┘  └──────────────────┘  └──────────────────┘
              │                    │                         │
              └────────────────────┴─────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────┐
                    │  Report Hidden Ports     │
                    │  - Port number           │
                    │  - Protocol (TCP/UDP)    │
                    │  - External tool output  │
                    └──────────────────────────┘
```

## File Structure and Responsibilities

### unhide-linux Components

| File | Lines | Primary Responsibility |
|------|-------|------------------------|
| unhide-linux.c | 856 | Main program logic, CLI parsing, checkps validation |
| unhide-linux-bruteforce.c | 277 | PID space exhaustion using vfork/pthread |
| unhide-linux-procfs.c | 454 | /proc-based detection (stat, chdir, opendir, readdir) |
| unhide-linux-syscall.c | 790 | System call-based detection (11 different syscalls) |
| unhide-linux-compound.c | 425 | Composite tests (quick, reverse) |
| unhide-linux.h | 166 | Constants, structures, function prototypes |

### unhide-tcp Components

| File | Lines | Primary Responsibility |
|------|-------|------------------------|
| unhide-tcp.c | 577 | Main TCP/UDP scanning logic |
| unhide-tcp-fast.c | 220 | Fast scanning implementation |
| unhide-tcp.h | 64 | Port scanning constants and definitions |

### Shared Components

| File | Lines | Primary Responsibility |
|------|-------|------------------------|
| unhide-output.c | 208 | Unified output and logging system |
| unhide-output.h | 50 | Output function interfaces |

## Process State Machine

```
┌───────────┐
│   START   │
└─────┬─────┘
      │
      ▼
┌─────────────────┐
│  INIT PHASE     │
│  - Parse args   │
│  - Get maxpid   │
│  - Open log     │
└────┬────────────┘
     │
     ▼
┌─────────────────┐        NO    ┌──────────────┐
│ Tests Selected? │──────────────▶│ Show Usage   │
└────┬────────────┘               │ Exit(1)      │
     │ YES                        └──────────────┘
     ▼
┌─────────────────────────────────────────┐
│  SCANNING PHASE                         │
│  ┌───────────────────────────┐          │
│  │ For each test selected:   │          │
│  │  - Execute test function  │          │
│  │  - Call checkps() verify  │          │
│  │  - Report any findings    │          │
│  └───────────────────────────┘          │
└────┬────────────────────────────────────┘
     │
     ▼
┌─────────────────┐
│  CLEANUP PHASE  │
│  - Free memory  │
│  - Close log    │
│  - Set exit code│
└────┬────────────┘
     │
     ▼
┌─────────────────┐
│  EXIT           │
│  Code: 0 or 1   │
└─────────────────┘
```

## Key Design Patterns

### 1. Strategy Pattern (Detection Methods)

Each detection method is encapsulated in its own function:
- `checkproc()` - stat strategy
- `checkchdir()` - chdir strategy
- `checkgetpriority()` - getpriority strategy
- etc.

All methods follow the same pattern:
```c
void check_method(void) {
    for (pid = 1; pid <= maxpid; pid++) {
        if (method_detects_process(pid)) {
            if (!checkps(pid, flags)) {
                printbadpid(pid);
            }
        }
    }
}
```

### 2. Template Method Pattern (Test Execution)

```c
struct tab_test_t {
    int todo;
    void (*func)(void);
};

// Test table
struct tab_test_t tab_test[MAX_TESTNUM];

// Execute selected tests
for (i = 0; i < MAX_TESTNUM; i++) {
    if (tab_test[i].todo) {
        tab_test[i].func();
    }
}
```

### 3. Facade Pattern (Output System)

`unhide-output.c` provides a unified interface:
- `msgln()` - Standard messages
- `warnln()` - Warnings
- `die()` - Fatal errors

All functions handle both console output and log file writing.

### 4. Double-Checked Locking (Brute Force)

```c
// First scan
for (i = 0; i < maxpid; i++) {
    allocate_pid();
    mark_in_array1();
}

// Second scan
for (i = 0; i < maxpid; i++) {
    allocate_pid();
    mark_in_array2();
}

// Report only PIDs missed in both scans
for (i = 0; i < maxpid; i++) {
    if (array1[i] && array2[i]) {
        report_hidden(i);
    }
}
```

## Security Architecture

### Static Linking
- **Why**: Prevents compromised system libraries from hiding processes
- **Implementation**: `gcc --static` flag in build_all.sh
- **Trade-off**: Larger binary size vs. better security

### Privilege Requirements
- **Required**: root/sudo access
- **Reason**: Access to all /proc entries and privileged system calls
- **Validation**: Check at runtime, die() if insufficient privileges

### Race Condition Mitigation

```c
// Check before
stat_before = stat("/proc/[pid]");
if (stat_before != 0) continue;

// Validate with ps
if (checkps(pid)) continue;

// Check after
stat_after = stat("/proc/[pid]");
if (stat_after != 0) continue;

// Only report if consistent
printbadpid(pid);
```

## Performance Considerations

### Optimization Techniques

1. **Early Exit**: checkps() returns immediately when process found
2. **Bit Flags**: Use bitmasks for check types (PS_PROC, PS_THREAD, PS_MORE)
3. **Array Pre-allocation**: Allocate PID arrays once in brute force
4. **Pipe Optimization**: Use stdbuf for unbuffered pipes
5. **Skip Self**: Always skip mypid to avoid self-detection

### Time Complexity

| Test | Complexity | Notes |
|------|------------|-------|
| checkproc | O(maxpid) | Linear scan with stat() |
| checkgetpriority | O(maxpid) | Linear scan with syscall |
| brute | O(maxpid) | 2x if double-check enabled |
| quick | O(maxpid × 11) | 11 tests per PID |
| reverse | O(n) | n = number of processes in ps |

### Memory Usage

| Component | Memory |
|-----------|--------|
| allpids array | 4 × maxpid bytes |
| allpids2 array | 4 × maxpid bytes (if -d) |
| ps output buffer | Dynamic, typically < 1MB |
| proc_tasks (rb) | 8 × maxpid bytes |

---

**Document Version**: 1.0  
**Last Updated**: December 2024  
**Based on Code Version**: 20240509
