---
title: Android 65536问题探究
date: 2017-07-30 10:17:32
categories: Android
---
### 问题出现

&emsp;&emsp;随着现在apk功能的增加，工程的源码会不断增加，再加上引用的第三方库，导致工程中的方法数急剧增加。默认情况下打包工具会把所有定义及引用的方法都记录到一个dex文件中，当打包工具检测到一个dex文件中的方法数超过65536时就会抛出异常，从而导致打包失败。对应的代码如下（platform_dalvik-master\dx\src\com\android\dx\merge\Dexmerge.java，android4.2源码)：
<!--more-->

```java
private void mergeMethodIds() {
    new IdMerger<MethodId>(idsDefsOut) {
        @Override TableOfContents.Section getSection(TableOfContents tableOfContents) {
            return tableOfContents.methodIds;
        }
        @Override MethodId read(Dex.Section in, IndexMap indexMap, int index) {
            return indexMap.adjust(in.readMethodId());
        }
        @Override void updateIndex(int offset, IndexMap indexMap, int oldIndex, int newIndex) {
            if (newIndex < 0 || newIndex > 0xffff) {
                throw new IllegalArgumentException("method ID not in [0, 0xffff]: " + newIndex);
            }
            indexMap.methodIds[oldIndex] = (short) newIndex;
        }
        @Override void write(MethodId methodId) {
            methodId.writeTo(idsDefsOut);
        }
    }.mergeSorted();
}
```
从第10行可以看出，当心方法索引大于0xffff时就会抛出异常，及65536问题。

### 问题分析
#### 方法的统计
&emsp;&emsp;文件中的方法数超过64K时打包会失败，那么这64K的方法都包含哪些呢？

&emsp;&emsp;首先，肯定包括工程源码（包括第三方jar）中的所有方法，不管方法的access_flags（public、private、static、final等）是啥。而且父类中的方法被子类每override一次计数时就会多一个。

&emsp;&emsp;其次，那些在源码中那个引用，但最终不会打包进apk的方法也会被统计，比如引用的android sdk中的方法。如果一个apk中只定义了1个方法，但是引用了其他sdk中的64K个方法，那么最终打包也会出现方法数超过64K的问题。

#### 山穷水尽疑无路
&emsp;&emsp;根据前面的源码可以看到，buildTool的源码中直接就判断方法的id数是否超过了0xffff。但是为什么要加这个限制呢？首先想到的是dex文件对格式的要求。dex文件在代码中是使用C语言的struct来定义的（paltform_dalvik-master\libdex\DexFile.h）：

```c
struct DexFile {
    /* directly-mapped "opt" header */
    const DexOptHeader* pOptHeader;
    /* pointers to directly-mapped structs and arrays in base DEX */
    const DexHeader*    pHeader;
    const DexStringId*  pStringIds;
    const DexTypeId*    pTypeIds;
    const DexFieldId*   pFieldIds;
    const DexMethodId*  pMethodIds;
    const DexProtoId*   pProtoIds;
    const DexClassDef*  pClassDefs;
    const DexLink*      pLinkData;
    /*
     * These are mapped out of the "auxillary" section, and may not be
     * included in the file.
     */
    const DexClassLookup* pClassLookup;
    const void*         pRegisterMapPool;       // RegisterMapClassPool
    /* points to start of DEX file data */
    const u1*           baseAddr;
    /* track memory overhead for auxillary structures */
    int                 overhead;
    /* additional app-specific data structures associated with the DEX */
    //void*               auxData;
};
```
其中dex文件的头部DexHeader结构： 

```c
struct DexHeader {
    u1  magic[8];           /* includes version number */
    u4  checksum;           /* adler32 checksum */
    u1  signature[kSHA1DigestLen]; /* SHA-1 hash */
    u4  fileSize;           /* length of entire file */
    u4  headerSize;         /* offset to start of next section */
    u4  endianTag;
    u4  linkSize;
    u4  linkOff;
    u4  mapOff;
    u4  stringIdsSize;
    u4  stringIdsOff;
    u4  typeIdsSize;
    u4  typeIdsOff;
    u4  protoIdsSize;
    u4  protoIdsOff;
    u4  fieldIdsSize;
    u4  fieldIdsOff;
    u4  methodIdsSize;
    u4  methodIdsOff;
    u4  classDefsSize;
    u4  classDefsOff;
    u4  dataSize;
    u4  dataOff;
};
```
第19行可以看到一个methodIdsSize字段，表示dex文件中的方法数，它是一个u4类型的，这个类型是一个无符号的32位整型，定义如下（platform_dalvik-master\vm\Common.h)： 

```c
typedef uint8_t             u1;
typedef uint16_t            u2;
typedef uint32_t            u4;
typedef uint64_t            u8;
typedef int8_t              s1;
typedef int16_t             s2;
typedef int32_t             s4;
typedef int64_t             s8;
```
因此dex文件中最大的方法数应该是$2^{32}$，而不是$2^{16}$。那到底是哪里加的限制呢？
#### 雄关漫道真如铁，而今迈步从头
&emsp;&emsp;理论上Dex文件对方法数的限制是$2^{32}$个，可是为什么打包工具检测到超过$2^{16}$就报错呢？打包工具肯定不会说：我就是这么任性，你咬我啊。一定是有什么原因才导致打包工具强加了这个限制。对于分析问题陷入迷惘的时候，福尔摩斯教给我们一个放之四海皆准的方法：排除法。既然dex文件本身没有问题，那么应该是在使用的时候有问题。dex文件和class文件差不多，都是一堆字节码，dalvik虚拟机会加载dex文件并执行里面的字节码。所以可以看一下dalvik虚拟机的源码，在platform_dalvik-master\vm\mterp\out下面有几对汇编和cpp源码。这些源码是在不同平台上对字节码的处理，原理都是差不多的，所以就分析InterpC-portable.cpp文件，其中有个叫dvmInterpretPortable的函数，这是实际处理字节码的地方。
##### dvmInterpretPortable函数分析
&emsp;&emsp;如果对这部分不感兴趣，可以直接跳到下一节。
 &emsp;&emsp;首先先看三个宏定义： 
 
 ```cpp
# define H(_op)             &&op_##_op
# define HANDLE_OPCODE(_op) op_##_op:
#define GOTO_TARGET(_target, ...) _target:
 ```
 
 &emsp;&emsp;这三个宏中后面俩其实就是根据参数来定义lable的，lable一般都是为了goto语句跳转用的。在dvmInterpretPortable函数中就使用了这两个宏定义了一系列的lable，不过我们目前只关心函数调用相关的地方：
 
 ```cpp
 /* File: c/OP_INVOKE_VIRTUAL.cpp */
HANDLE_OPCODE(OP_INVOKE_VIRTUAL /*vB, {vD, vE, vF, vG, vA}, meth@CCCC*/)
    GOTO_invoke(invokeVirtual, false);
OP_END
/* File: c/OP_INVOKE_SUPER.cpp */
HANDLE_OPCODE(OP_INVOKE_SUPER /*vB, {vD, vE, vF, vG, vA}, meth@CCCC*/)
    GOTO_invoke(invokeSuper, false);
OP_END
/* File: c/OP_INVOKE_DIRECT.cpp */
HANDLE_OPCODE(OP_INVOKE_DIRECT /*vB, {vD, vE, vF, vG, vA}, meth@CCCC*/)
    GOTO_invoke(invokeDirect, false);
OP_END
/* File: c/OP_INVOKE_STATIC.cpp */
HANDLE_OPCODE(OP_INVOKE_STATIC /*vB, {vD, vE, vF, vG, vA}, meth@CCCC*/)
    GOTO_invoke(invokeStatic, false);
OP_END
/* File: c/OP_INVOKE_INTERFACE.cpp */
HANDLE_OPCODE(OP_INVOKE_INTERFACE /*vB, {vD, vE, vF, vG, vA}, meth@CCCC*/)
    GOTO_invoke(invokeInterface, false);
OP_END
 ```
 &emsp;&emsp;我们只看OP\_INVOKE\_DIRECT，其他的几个也是相似的。前面说了HANDLE\_OPCODE是定义一个lable的，那么肯定有一个goto语句来跳到这个地方，在哪里呢？在dvmInterpretPortable函数中有一句话：`FINISH(0);`，FINISH也是一个宏，定义如下： 
 
 ```cpp
 # define FINISH(_offset) {                                                  \
    ADJUST_PC(_offset);                                                 \
    inst = FETCH(0);                                                    \
    if (self->interpBreak.ctl.subMode) {                                \
        dvmCheckBefore(pc, fp, self);                                   \
    }                                                                   \
    goto *handlerTable[INST_INST(inst)];                                \
}
 ```
 &emsp;&emsp;可以看到里面有个goto语句，但是这个handlerTable是个啥呢？goto后面应该是个标号，那么handlerTable应该是一个标号地址的数组或指针，之所以说handlerTable的元素是标号的地址，是因为它前面有一个星号（*）,了解C语言的应该知道，它是取一个地址里面的内容的，在这里这个内容就是标号。口说无凭啊，我们来看一下代码，还是dvmInterpretPortable函数中，有如下的一句： 
 
 ```cpp
 DEFINE_GOTO_TABLE(handlerTable);
 ```
 &emsp;&emsp;DEFINE\_GOTO\_TABLE，在C和C++中一看到这种大写的基本上就是宏。它的定义如下： 

```cpp
/*
 * Macro used to generate a computed goto table for use in implementing
 * an interpreter in C.
 */
#define DEFINE_GOTO_TABLE(_name) \
    static const void* _name[kNumPackedOpcodes] = {                      \
        /* BEGIN(libdex-goto-table); GENERATED AUTOMATICALLY BY opcode-gen */ \
        H(OP_NOP),                                                            \
        H(OP_MOVE),                                                           \
        /*此次省略N行*/                                                       \
        H(OP_INVOKE_VIRTUAL),                                                 \
        H(OP_INVOKE_SUPER),                                                   \
        H(OP_INVOKE_DIRECT),                                                  \
        H(OP_INVOKE_STATIC),                                                  \
        H(OP_INVOKE_INTERFACE),                                               \
    }
```
&emsp;&emsp;这里确实是一个数组，元素是一个void型的指针，那这个指针是啥呢？它是通过H宏定义的，结合前面H宏的定义，可以知道，这些元素就是dalvik虚拟机指令前加op\_定义的标号的地址。

&emsp;&emsp;为了防止信息量太多记不住，先小结一下： 

1. 在dvmInterpretPortable函数中，HANDLE\_OPCODE宏定义了一个形式是“op\_虚拟机指令”的标号
2. DEFINE\_GOTO\_TABLE宏定义了一个数组，其元素是形式为“op_虚拟机指令”的标号地址
3. FINISH(0)相当于插入了一段“goto op\_虚拟机指令”的代码
4. 3中的代码会导致跳至1中定义的标号处去执行

&emsp;&emsp;上面讲到代码是如何执行到dalvik虚拟机指令所定义的标号处的，对于invoke_direct 指令，其对标号(`HANDLE_OPCODE(OP_INVOKE_DIRECT /*vB, {vD, vE, vF, vG, vA}, meth@CCCC*/)`)对应的代码如下，只有一句：

```cpp
GOTO_invoke(invokeDirect, false);
```
&emsp;&emsp; 又是一个宏，其定义如下： 

```cpp
#define GOTO_invoke(_target, _methodCallRange)                              \
    do {                                                                    \
        methodCallRange = _methodCallRange;                                 \
        goto _target;                                                       \
    } while(false)

```
&emsp;&emsp;其主要代码就是一个goto语句，对于invoke\_direct指令，这个\_target就是invokeDirect，因此invoke\_direct指令标号处的代码就相当于：goto invokeDirect。
#### 柳暗花明又一村
&emsp;&emsp;上面分析了一堆，其实就是为了找到invok\_direct指令调用的地方，即invokeDirect。
&emsp;&emsp;首先先看一个宏：

```cpp
#define GOTO_TARGET(_target, ...) _target:
```
就是简单的定义一个标号，invokeDirect标号既是通过这个宏来定义的，如下：

```cpp
GOTO_TARGET(invokeDirect, bool methodCallRange)
    {
        u2 thisReg;
        EXPORT_PC();
        vsrc1 = INST_AA(inst);      /* AA (count) or BA (count + arg 5) */
        ref = FETCH(1);             /* method ref */
        vdst = FETCH(2);            /* 4 regs -or- first reg */
        if (methodCallRange) {
            ILOGV("|invoke-direct-range args=%d @0x%04x {regs=v%d-v%d}",
                vsrc1, ref, vdst, vdst+vsrc1-1);
            thisReg = vdst;
        } else {
            ILOGV("|invoke-direct args=%d @0x%04x {regs=0x%04x %x}",
                vsrc1 >> 4, ref, vdst, vsrc1 & 0x0f);
            thisReg = vdst & 0x0f;
        }
        if (!checkForNull((Object*) GET_REGISTER(thisReg)))
            GOTO_exceptionThrown();
        methodToCall = dvmDexGetResolvedMethod(methodClassDex, ref);
        if (methodToCall == NULL) {
            methodToCall = dvmResolveMethod(curMethod->clazz, ref,
                            METHOD_DIRECT);
            if (methodToCall == NULL) {
                ILOGV("+ unknown direct method");     // should be impossible
                GOTO_exceptionThrown();
            }
        }
        GOTO_invokeMethod(methodCallRange, methodToCall, vsrc1, vdst);
    }
GOTO_TARGET_END
```

&emsp;&emsp;先看一下dvmDexGetResolvedMethod方法调用（第19行），这个方法就是到dex文件中根据一个索引（ref的值）去查找相应的method。前面说了，dex文件中最多可查询的方法数是$2^{32}$， 远大于65536，因此怀疑是ref这个值限制了这个个数，不过ref是一个u4类型的值，也是32位无符号整数，似乎也排除了。其实不然，来看一下它的赋值那一行，FETCH这个宏就是从pc计数器的某个位置取字节码的，这个是从offset为1的位置取一个数（屌丝程序猿会问为啥是从1开始取，而不是0呢，嗯，请移步到前面说的FINISH宏），关键是后面的注释，它说取到的这个数是一个方法的引用。这个时候你的心应该开始荡漾起来了：莫非这个引用是一个16位的数？！答案是肯定的。证据呢？去看dalvik虚拟机的invoke-direct指令（应该是一系列指令，叫invoke-kind，[虚拟机指令集]("https://source.android.com/devices/tech/dalvik/dalvik-bytecode.html"))，官网有详细说明，不过有可能被墙，所以这里只简单说明一下。函数调用的指令形式如下：
invoke-kind {vC, vD, vE, vF, vG}, meth@BBBB
那个kind就是表示不同类型的函数调用，比如direct、super、virtual等，主要看meth@BBBB这个，这个是invoke-kind指令的一个操作数，表示一个方法的索引，用来索引dex文件中的方法，@后面的BBBB表示是这个操作数的位数，一个大写的字母表示4位，因此这个方法的索引就是16位的，只能索引65536个方法！当dex文件中的方法数超过65536时后面的方法根本就索引不到，导致函数调用不了，这在运行的时候肯定会出错的，因此打包的时候检查到一个dex文件方法数超过65536时就会打包失败，早发现早治疗，谷歌表示这锅他不背。至此真相就大白了。

### 方法数限制之外
&emsp;&emsp;一个dex文件中方法数超过65536 时会导致打包失败的问题分析完了，不过你要是以为只有方法数有这个限制你就太天真了。成员变量、方法签名数等也有这个限制！原因都是差不多的，就不分析了。
