---
title: Linux Bus error问题研究
date: 2025-12-09 20:53:00 +0800
categories: [Linux]
tags: [Linux]
---
<script async src="https://busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>

<link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.3.1/css/all.css" integrity="sha384-mzrmE5qonljUremFsqc01SB46JvROS7bZs3IO2EmfFsd15uHvIt+Y8vEf7N7fWAU" crossorigin="anonymous">

<p align="right"><i class="fa fa-eye"></i> 阅读次数: <span id="busuanzi_value_page_pv"><i class="fa fa-spinner fa-spin"></i></span></p>


进程崩溃，屏幕或终端上打印`bus error` 的信息，本质上是进程接收到了内核发送的`SIGBUS`信号，进程终止。

## SIGBUS 发送流程
1. 内核把 **SIGBUS 信号** 直接发送给目标进程.
2. 一般进程没有自定义 signal handler，因此会执行glibc 提供的默认信号处理机制。
3. 进程被终止。
4. shell 检测进程退出状态，将信号转化为字符串输出到终端 也就是 Bus error 

## SIGBUS 类型
SIGBUS的类型定义在内核源码的`include/uapi/asm-generic/siginfo.h`中
```c
#define BUS_ADRALN	1	/* invalid address alignment */
#define BUS_ADRERR	2	/* non-existent physical address */
#define BUS_OBJERR	3	/* object specific hardware error */
/* hardware memory error consumed on a machine check: action required */
#define BUS_MCEERR_AR	4
/* hardware memory error detected in process but not consumed: action optional*/
#define BUS_MCEERR_AO	5
```

### BUS_ADRALN

BUS_ADRALN是指对齐错误访问内存。x86平台默认支持非对齐内存访问，只有强制开启内存对齐检查，才会触发SIGBUS。以下面代码为例：
```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <ucontext.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
//使能对齐检测,非对齐内存访问 导致bus error
 #if defined(__GNUC__)
 # if defined(__i386__)
     __asm__("pushf\n orl $0x40000,(%esp) \n popf");
 # elif defined(__x86_64__)
     __asm__("pushf\n orl $0x40000,(%rsp) \n popf");
 # endif
 #endif
    short array[16];
    int * p = (int *) &array[1];
    *p = 1;
    return 0;
}
```
在64位下运行报错是段错误，在32位下运行报错是Bus error。
如果不开启对齐检测，程序是可以正常运行的。  此demo 是可以触发core 文件dump 的，可以定位到问题代码。

### BUS_ADRERR
BUS_ADRERR 是指访问了不存在的物理内存。虚拟地址映射存在，但背后没有合法物理页，以下面代码为例子，在x86平台32位和64为运行效果是一样的。
```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <ucontext.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>


int main() {
    int fd = open("test_file", O_RDWR | O_CREAT, 0666);
    ftruncate(fd, 4096);   // 只给1页

    size_t map_len = 8192; // 映射2页
    char *p = mmap(NULL, map_len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    
    printf("Write in mapped but beyond file...\n");
    p[5000] = 'X';  // → 100% SIGBUS
    return 0;
}
```

设置文件大小为4096字节，但是内存映射8192字节，当访问5000字节偏移的时候 就会触发SIGBUS.该程序运行结果如下：

![](/assets/image/Pasted%20image%2020251203145536.png)


通过gdb 调试可以定位到问题代码：

![](/assets/image/Pasted%20image%2020251203145641.png)

### BUS_OBJERR

BUS_OBJERR 是指对象特定硬件错误，软件无法复现，在linux 3.14.44 的内核源码中搜索BUS_OBJERR 字符串，结果如下：

![](/assets/image/Pasted%20image%2020251203134403.png)

表明x86平台不会触发此类型的SIGBUS。

### BUS_MCEERR_AR 和 BUS_MCEERR_AO

这两个类型完全跟硬件有关系。表示硬件内存存在错误，无法通过软件demo 来复现此类型SIGBUS。



## 自定义信号处理函数

可以自定义SIGBUS的信号处理函数，来打印SIGBUS 类型以及调用栈等等，这样有个问题就是不会触发core dump了。

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <ucontext.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#include <signal.h>
#include <execinfo.h>


void print_bt() {
    void *bt[20];
    int n = backtrace(bt, 20);
    backtrace_symbols_fd(bt, n, STDERR_FILENO);
}

void sigbus_handler(int sig, siginfo_t *si, void *unused) {
    printf("Caught SIGBUS: si_addr=%p, si_code=%d\n", si->si_addr, si->si_code);

    // si_code 可以判断具体原因：
    // BUS_ADRALN   = 对齐错误
    // BUS_ADRERR   = 非法物理地址
    // BUS_OBJERR   = 设备/硬件错误
    // BUS_MCEERR_AR   = 设备/硬件错误
    // BUS_MCEERR_AO   = 设备/硬件错误
    switch(si->si_code) {
        case BUS_ADRALN: printf("Alignment error\n"); break;
        case BUS_ADRERR: printf("Nonexistent physical address\n"); break;
        case BUS_OBJERR: 
        case 4: 
        case 5: printf("Hardware / device error\n"); break;
        default: printf("Unknown cause\n"); break;
    }
    
    print_bt();
    exit(1); // 可以选择终止程序
}

int main() {
    struct sigaction sa;
    sa.sa_sigaction = sigbus_handler;
    sa.sa_flags = SA_SIGINFO  | SA_NODEFER | SA_RESTART;
    sigemptyset(&sa.sa_mask);

    sigaction(SIGBUS, &sa, NULL);


    int fd = open("test_file", O_RDWR | O_CREAT, 0666);
    ftruncate(fd, 4096);   // 只给1页

    size_t map_len = 8192; // 映射2页
    char *p = mmap(NULL, map_len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    
    printf("Write in mapped but beyond file...\n");
    p[5000] = 'X';  // → 100% SIGBUS
    return 0;
}
```
