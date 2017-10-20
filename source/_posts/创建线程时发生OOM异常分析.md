---
title: 创建线程时发生OOM异常分析

date: 2017-09-18 21:50:57
categories: Android
---
&emsp;&emsp;最近我们应用线上出现了一个问题：创建线程时发生OOM异常导致崩溃。领导让深入分析一下，看看是什么原因。既然要深入分析，那就只有一条路了：看源码。以下分析时基于Android 4.0.4源码。

### 1、Thread对象是如何和线程关联的
&emsp;&emsp;我们知道new一个Thread对象就会创建一个线程，那这个是怎么实现的呢？看Thread源码发现，当new一个Thread对象时并没有创建线程，只是一些变量赋值（Thread的构造方法会调用create方法）。
其实，线程的创建是在调用Thread对象的start方法是发生的，start方法的源码如下：

```java
public synchronized void start() {
        if (hasBeenStarted) {
            throw new IllegalThreadStateException("Thread already started."); // TODO Externalize?
        }

        hasBeenStarted = true;

        VMThread.create(this, stackSize);
    }
```

<!--more-->
通过调用VMThread的create方法，该方法是一个native方法，实现在/dalvik/vm/native/java\_lang\_VMThread.cpp文件中，如下：

```cpp
static void Dalvik_java_lang_VMThread_create(const u4* args, JValue* pResult)
{
    Object* threadObj = (Object*) args[0];
    s8 stackSize = GET_ARG_LONG(args, 1);

    /* copying collector will pin threadObj for us since it was an argument */
    dvmCreateInterpThread(threadObj, (int) stackSize);
    RETURN_VOID();
}
```

该方法又调用了dvmCreateInterpThread方法，这个方法定义在/dalvik/vm/Thread.cpp文件中，如下：

```cpp
bool dvmCreateInterpThread(Object* threadObj, int reqStackSize)
{
//省略部分代码
int cc = pthread_create(&threadHandle, &threadAttr, interpThreadStart,
                            newThread);
    dvmChangeStatus(self, oldStatus);

    if (cc != 0) {
        /*
         * Failure generally indicates that we have exceeded system
         * resource limits.  VirtualMachineError is probably too severe,
         * so use OutOfMemoryError.
         */
        LOGE("Thread creation failed (err=%s)", strerror(errno));

        dvmSetFieldObject(threadObj, gDvm.offJavaLangThread_vmThread, NULL);

        dvmThrowOutOfMemoryError("thread creation failed");
        goto fail;
   }
   //省略部分代码
}
```

dvmCreateInterpThread最终是调用pthread\_create方法来创建线程的，当pthread\_create方法返回错误时（cc不等于0）就会打印出错信息，并且会抛出一个OOM的异常，这个OOM最终会被throw到Java层引起崩溃。pthread\_create定义在/bionic/libc/bionic/pthread.c文件中，如下：

```c
int pthread_create(pthread_t *thread_out, pthread_attr_t const * attr,
                   void *(*start_routine)(void *), void * arg)
{
pthread_internal_t * thread;
//....
thread = _pthread_internal_alloc();
//省略无关代码
if (!attr->stack_base) {
        stack = mkstack(stackSize, attr->guard_size);
        if(stack == NULL) {
            _pthread_internal_free(thread);
            return ENOMEM;
        }
        madestack = 1;
    } else {
        stack = attr->stack_base;
    }
//省略无关代码
```

该方法先为thread的结构体分配一块内存，然后调用了mkstack方法为当前thread分配一块栈空间，该方法定义如下：

```c
static void *mkstack(size_t size, size_t guard_size)
{
    void * stack;

    pthread_mutex_lock(&mmap_lock);

    stack = mmap((void *)gStackBase, size,
                 PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS | MAP_NORESERVE,
                 -1, 0);

    if(stack == MAP_FAILED) {
        stack = NULL;
        goto done;
    }

    if(mprotect(stack, guard_size, PROT_NONE)){
        munmap(stack, size);
        stack = NULL;
        goto done;
    }

done:
    pthread_mutex_unlock(&mmap_lock);
    return stack;
}
```

mkstack调用mmap方法为当前线程在进程的虚拟地址空间分配一块空间（默认情况下该空间大小约为1M）。

从上面分析可知Java层的Thread对象时如何与native层的线程关联起来的，说白了就是Thread是通过JNI对native线程及其它操作的一个封装。

### 2、问题分析

&emsp;&emsp;当时面临两个问题需要高明白：

1. 为什么发生OOM
2. 为什么上传到服务器上的崩溃没有native层的

&emsp;&emsp;这两个问题之后会解答，现在先对代码进行分析。

&emsp;&emsp;从pthread\_create方法中可以看出，调用\_pthread\_internal\_alloc和mkstack方法失败时都表示发生了OOM（返回值为ENOMEM），我刚开怀疑的是mkstack，因为另外一个函数分配的内存比较小，出错的概率也就比较小，事实证明也就是这个函数调用失败了。在mkstack函数中调用了mmap，最根本的原因就在于这个mmap方法调用失败了。

&emsp;&emsp;mmap是用于在进程虚拟地址空间分配一段指定大小的连续空间，对于32位系统来说，进程的虚拟地址空间为4g，一般系统会占用1g，剩余3g是给应用使用的，这3g包括framework和应用的apk、系统及应用的so、预先分配的堆空间等等，实际当一个应用刚运行的时候可能已经分配大于1个g的空间了，所以给应用用来分配的一般不到2g。当不断创建新线程时，就会不断分配虚拟地址空间，很明显，当线程都到一定数量之后虚拟地址空间就会被耗尽，从而引发OOM。

&emsp;&emsp;为了验证上面的说法，我写了一个demo，通过JNI在native层不断创建线程，当pthread_create返回值不为0时打印一句log，并退出循环。代码如下：

```cpp
int create_thread() {
    while(true) {
            count++;
             pthread_t pt;
             int cc = pthread_create(&pt, NULL, &thread_fun, (void *) NULL);
            if(count % 100 == 0) {
                sleep(1);
            }
            if(cc != 0) {
                __android_log_print(ANDROID_LOG_WARN, "create_thread", "cc=%d,count=%d", cc, count);
                return cc;
            }
        }
 }
```

&emsp;&emsp;当看到由于线程创建失败打印的信息时，就从设备中把/proc/{pid}/maps文件pull出来，这个文件记录了当前进程虚拟地址空间的映射情况。到此可以回答刚开始提出的两个问题：

&emsp;&emsp;对于第一个问题，时由于线程创建太多会导致虚拟地址空间耗尽，此时虚拟机在Java层抛出了一个OOM异常。

&emsp;&emsp;对于第二个问题，从demo中的打印可以看出，当pthread\_create失败时仅仅是返回一个不为0的返回值，程序完全可以忽略这个值，因此并不会导致崩溃。

### 3、来自系统的恶意
&emsp;&emsp;上面分析的时一般的问题原因，demo也是在一般的设备上测试的，我们的应用刚启动时也就创建了100个线程左右，应该不会发生OOM才对。但是当我把我的demo放到有问题的设备上测试时，发现maps文件的虚拟地址空间才映射到0x7d398000。这离系统分配的栈空间的起始地址：be9f9000（大致这个位置）还早着呢，怎么就失败了呢？

&emsp;&emsp;首先想到的时这个设备用于应用分配的空间有限制。然后就开始验证：当pthread\_create方法失败时，在直接调用mmap方法分配了一个300M内存，发现并没有失败（看返回值），pull出来的maps文件也可以看出有一段300M的虚拟地址空间映射。那就说明还有其他原因。从pthread\_create返回失败时打印的log看出来，当前的进程值创建了600多个线程就失败了，而我在另外一个设备上创建了1300多，难道是系统对线程数有限制？于是就在设备上看了一下/proc/sys/kernel/threads-max文件，发现这个里面的数值是1371，即这个设备的系统中最多只能创建1371个线程，我查看了一下这个设备刚启动时候的线程数（ps -t | wc -l），发现有800多个，当在设备上启动几个应用之后很快就上升到1000多，这个时候如果在启动我们的应用就会有可能超过系统总的线程数，从而引发OOM。

&emsp;&emsp;这个问题最终也没有想到好的解决方法，只能降低自己应用的线程数，降低OOM发生的概率。