---
title: 关于fork函数
date: 2017-07-22 16:27:27
categories: Linux
---
<!-- <font face="微软雅黑" size=4>-->
## **1、写作缘由**
&emsp;&emsp;fork函数是用来创建子进程的，不过，长久以来让我迷惑不解的就是它的一次掉用两次返回，而且两次返回的结果还是不同的：父进程中返回的是子进程的ID，子进程中返回的是0。最近在看Linux 0.11的内核源码，突然犹如那个醍醐灌顶，拨云见日，一切豁然开朗啊。

<!--more-->

## **2、fork函数的定义**
&emsp;&emsp;这一切的一切还要从./init/mai.c这个文件开始说起，在开始的地方有这么一行：

```c
static inline _syscall0(int,fork)
```

\_syscall0时一个宏，定义在./include/unistd.h中的133行，如下：

```c
#define _syscall0(type,name) \
  type name(void) \
  { \
      long __res; \
      __asm__ volatile ("int $0x80" \   //系统调用
      : "=a" (__res) \    //把eax寄存器的值赋值给__res
      : "0" (__NR_##name)); \   //把__NR_##name放到0号寄存器，即eax
      if (__res >= 0) \
      return (type) __res; \
      errno = -__res; \
      return -1; \
 } 
```

AT&T嵌入汇编格式[^atasm]

因此` static inline _syscall0(int,fork) `展开之后为：

```c
static inline int fork(void) 
  { 
      long __res; 
      __asm__ volatile ("int $0x80" 
      : "=a" (__res) 
      : "0" (__NR_fork)); 
      if (__res >= 0) 
      return (int) __res; 
      errno = -__res; 
      return -1; 
 }
```

&emsp;&emsp;这个就是fork函数的定义啊！其中，\_\_NR\_fork也是定义在./include/unistd.h中的，在62行：#define \_\_NR\_fork 2。这个函数就是去调用0x80号系统调用的2号功能。可是这个2号功能是咩呢？那就请看./include/linux/sys.h文件中的74行开始的地方：

```c
fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,
sys_write, sys_open, sys_close, sys_waitpid, sys_creat, sys_link,
sys_unlink, sys_execve, sys_chdir, sys_time, sys_mknod, sys_chmod,
sys_chown, sys_break, sys_stat, sys_lseek, sys_getpid, sys_mount,
sys_umount, sys_setuid, sys_getuid, sys_stime, sys_ptrace, sys_alarm,
sys_fstat, sys_pause, sys_utime, sys_stty, sys_gtty, sys_access,
sys_nice, sys_ftime, sys_sync, sys_kill, sys_rename, sys_mkdir,
sys_rmdir, sys_dup, sys_pipe, sys_times, sys_prof, sys_brk, sys_setgid,
sys_getgid, sys_signal, sys_geteuid, sys_getegid, sys_acct, sys_phys,
sys_lock, sys_ioctl, sys_fcntl, sys_mpx, sys_setpgid, sys_ulimit,
sys_uname, sys_umask, sys_chroot, sys_ustat, sys_dup2, sys_getppid,
sys_getpgrp, sys_setsid, sys_sigaction, sys_sgetmask, sys_ssetmask,
sys_setreuid,sys_setregid };
```

&emsp;&emsp;很容易看出这是一个数组，元素的类型是fn\_ptr,这个又是什么呢？那就去看./include/linux/sech.h文件的第38行：typedef int (*fn\_ptr)();可以看出fn\_ptr是一个函数指针，它所指向的是一个无参的返回类型为int型的函数。作为一个程序员，你应该很敏锐的发现这个功能号2对应的正好是sys\_fork在数组中的索引。没错，2号功能就是调用sys_fork函数。

## **3、古老的回忆**
### **3.1&emsp;一直没有搞明白的int 0x80指令**
&emsp;&emsp;上面fork函数的定义很简单，它最重要的地方就是`int $0x80`这个指令。我紧紧盯着它，“眼泪洒在记忆的门里”。以前学汇编的时候，书上说`int 0x80`是一个系统调用，这个指令还会取eax寄存器中的值作为功能号。那么，int指令到底干啥用的？说0x80号是系统调用（system\_call，定义在./kernel/system_\call.s中）到底是什么意思呢？

&emsp;&emsp;这就要从中断向量表（赵博士的书中有介绍）开始说起了，你可以把中断向量表想象成一个数组，数组的每一个元素是八个字节的，这八个字节中就包含了一个中断处理程序的地址，说人话就是一个函数的地址，至于这八个字节中的那些位表示地址，可以搜索“中断描述符”了解相关知识。 其实，这个0x80就是这个数组的一个索引。给这个0x80索引位置赋值是在./kernel/sched.c的第411行：

`set\_system\_gate(0x80, &system\_call);`

上面这个宏定义在./include/asm/system.h中，可以看到这个位置就是system\_call函数的地址。set\_system\_gate宏就不展开了，不过还是要说一下这个宏中的idt定义在./head.s中的232行。

&emsp;&emsp;中断向量表的作用就是在CPU收到一个中断请求时，到这个表里面找一个中断处理程序来做相应的处理，这些中断处理程序都是系统事先定义好的。

&emsp;&emsp;因此，int指令就是向CPU发送一个中断请求，而0x80就是告诉CPU调用中断向量表中的0x80（128）号元素指向的函数（中断处理程序）来处理这个请求。而这个0x80处的元素指向的函数就是system\_call，翻译过来就是“系统调用”。

### **3.2 &emsp;谜一样的系统调用功能号**

&emsp;&emsp;书中说，在系统调用之前，得先向eax寄存器写入一个功能号，当时只知道要这么做，但不知道为什么要这么做，而且这功能号和中断向量号（例如0x80）傻傻分不清楚。3.1节解释了中断向量号是怎么一回事，在2节说明了系统调用的功能号其实是一个数组的索引，其对应的元素是一个函数指针，即一个具体的功能。那么int指令、中断向量号和功能号是怎么联系起来的呢？

&emsp;&emsp;3.1节的最后说明了int和向量号是怎么联系起来的，即：int指令向CPU发送一个中断请求，CPU根据中断向量号找到中断处理程序的入口地址，然后从入口地址处开始执行命令。那功能号有什么用？先来看看system\_call方法的实现，在./kernel/system\_call.s文件中：

```asm
_system_call:
        cmpl $nr_system_calls-1,%eax
        ja bad_sys_call
        push %ds 
        push %es 
        push %fs 
        pushl %edx
        pushl %ecx              # push %ebx,%ecx,%edx as parameters
        pushl %ebx              # to the system call
        movl $0x10,%edx         # set up ds,es to kernel space
        mov %dx,%ds
        mov %dx,%es
        movl $0x17,%edx         # fs points to local data space
        mov %dx,%fs
        call _sys_call_table(,%eax,4)
        pushl %eax
        movl _current,%eax
        cmpl $0,state(%eax)             # state
        jne reschedule
        cmpl $0,counter(%eax)           # counter
        je reschedule
```

重点看一下第15行：

```asm
call \_sys\_call\_table(,%eax,4)
```

\_sys\_call\_table(,%eax,4)是一个寻址，相当于\_sys\_call\_table+%eax*4，而%eax存放的就是功能号2，这也解释了为什么先把功能号放到eax寄存器，乘4是因为每个指针有4个字节，所以前面的地址就成了\_sys\_call\_table+8了，这个就是sys.h中的sys\_call\_table数组中的第二项，所以这句就等价于：

```
call sys_fork
```

&emsp;&emsp;**总结一下：**int指令通过给CPU发送一个中断请求，该指令有一个操作数（0x80）指明了处理这个请求的中断处理程序入口地址在中断向量表中的位置，于是CPU从该入口地址处开始执行这段程序，这段程序会取出eax寄存器中的功能号，然后根据这个功能号到sys\_call\_table中获取一个函数指针，再调用这个函数指针指向的函数（即功能）。现在要分析的这个功能就是创建子进程：sys\_fork函数。

## **4、fork的两次返回**

### **4.1 &emsp;进程是怎么创建的**
&emsp;&emsp;根据上面的分析可知，调用fork函数最终会调用系统的sys\_fork函数来实现子进程的创建，那么就来看看这个函数干了什么：

```asm
_sys_fork:
        call _find_empty_process
        testl %eax,%eax
        js 1f
        push %gs
        pushl %esi
        pushl %edi
        pushl %ebp
        pushl %eax
        call _copy_process
        addl $20,%esp
1:      ret
```

重点看第2和第10行.

\_find\_empty\_process方法是找到一还未使用的进程好，保存到一个叫last\_pid的全局变量中，然后找到一个空闲的任务，并返回它的索引。

\_copy\_process函数的定义如下：

```c
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
                long ebx,long ecx,long edx,
                long fs,long es,long ds,
                long eip,long cs,long eflags,long esp,long ss)
{
        struct task_struct *p;
        int i;
        struct file *f;

        p = (struct task_struct *) get_free_page();
        if (!p)
                return -EAGAIN;
        task[nr] = p;
        //新进程就是当前进程的一份拷贝，所以他们刚开始时共享物理内容的，即共享代码和数据
        *p = *current;  /* NOTE! this doesn't copy the supervisor stack */
        p->state = TASK_UNINTERRUPTIBLE;
        //把find_empty_process方法中找到的进程id赋值给新进程
        p->pid = last_pid;
        //新进程的父进程id就是当前进程的id
        p->father = current->pid;
        //省略部分代码
        //eip就是sys_fork调用该函数时压入栈的eip值
        p->tss.eip = eip;
        //为什么子进程中的返回值是0？答案就在这里
        p->tss.eax = 0;
        //省略部分代码
        //cs就是sys_for调用该函数数压入栈的cs值
        p->tss.cs = cs & 0xffff;
        
        return last_pid;
```
调用get\_free\_page方法分配一页内存，然后转为task\_struct结构，这个task\_struct就是进程的数据结构，内存中的一个task\_struct就代表一个进程。所以创建一个进程对系统来说就是创建一个task_struct结构。

`*p = *current`这句就说明子进程是当前进程的拷贝，即他们共享物理内存，包括代码和数据。由于代码是只读的，所以会一直共享，数据是可写的，所当数据页被写时就会发生了著名的写时复制（Copy-On-Write）。

`p->tss.exa=0`这句就是把子进程的返回值强制设置成了0。

### **4.2&emsp;第二次返回时如何发生的**
&emsp;&emsp;那为什么会两次返回呢？这还要想回到sys\_fork函数的定义中，执行`call _copy_process`时，发生了函数调用，此时cs和eip的值会被压栈，cs和eip正指向下一条指令，即`addl $20,%esp`。在copy\_process函数中，这两个值被赋值给了新进程的tss.cs和tss.eip。当新进程创建完成并设置为RUNNING状态之后，新进程就可以被调度了，这里假设CPU先调度执行父进程（即当前进程）：

第一次返回（父进程返回）：由于父进程正在执行copy\_process函数，因此这个函数的返回值就是最终fork的返回值，而这个函数的返回值就是last\_pid，即通过find\_empty\_process方法分配的新进程id（即子进程id）。返回路径：copy\_process->sys\_fork->system\_call->fork。

第二次返回（子进程返回）：首先来看子进程被调度时会从哪里开始执行。决定从哪里执行的就是cs和eip寄存器的值，因为CPU是根据cs和eip的值来取下一条指令的。当进程被调度时，会把进程的tss.cs和tss.eip中的值恢复到cs和eip中，根据上面代码中加的注释（22和27行）知道，子进程的tss.cs和tss.eip中的值就是调用copy\_process时压入的cs和eip，即他们指向了第sys\_for函数的`addl $20,%esp`，因此子进程就从这一行开始执行，然后就按照sys\_fork->system\_call->fork这样的路径返回，这就是fork函数的第二次返回，它的值就是从子进程的tss.eax回复到eax寄存器中的值：0。


[^atasm]: \_asm\_("汇编语句": [汇编代码执行完毕后输出的寄存器 ]: [在汇编代码执行前需要输入的参数 ]:  [代码执行过程中会被修改的寄存器)]; "a" "b" "c" "d"分别表示寄存器eax, ebx, ecx, edx ,"S"和"D"表示寄存器esi,edi,串操作中用到 ,"r" 表示任何寄存器 ,"n"表示与上面位置相同的同一寄存器，即寄存器在上三个[]中出现的位置，从0开始计数。
<!--</font>-->