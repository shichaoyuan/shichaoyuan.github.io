---
layout: post
title:  "About Memory Barrier"
date:   2013-08-10 17:20:01
tags: concurrency
---

## 引言

什么是Memory Barrier？stackoverflow中有一个经典的[回答](http://stackoverflow.com/questions/286629/what-is-a-memory-fence)：

> For performance gains modern CPUs often execute instructions out of order to make maximum use of the available silicon (including memory read/writes). Because the hardware enforces instructions integrity you never notice this in a single thread of execution. However for multiple threads or environments with volatile memory (memory mapped I/O for example) this can lead to unpredictable behavior.

> A memory fence/barrier is a class of instructions that mean memory read/writes occur in the order you expect. For example a 'full fence' means all read/writes before the fence are comitted before those after the fence.

> Note memory fences are a hardware concept. In higher level languages we are used to dealing with mutexes and semaphores - these may well be implemented using memory fences at the low level and explicit use of memory barriers are not necessary. Use of memory barriers requires a careful study of the hardware architecture and more commonly found in device drivers than application code.

> The CPU reordering is different from compiler optimisations - although the artefacts can be similar. You need to take separate measures to stop the compiler reordering your instructions if that may cause undesirable behaviour (e.g. use of the volatile keyword in C).

从高级语言编程的角度，理解与掌握Memory Barrier是容易的，但是对于硬件角度的Memory Barrier却有很多疑惑。所有笔者尝试通过本文自上而下将整个story串起来。

## 幼稚的例子

这个例子的思路来在一个带有cancel功能的GUI程序。（这种方式绝对的幼稚，但是本文的目的不在于cancel功能的实现，所以在下文也不会完善该功能，所做的改动仅仅是为了展示memory barrier）


```java
public class MemoryBarrierDemo {

        private static volatile boolean cancel;

        public static void main(String[] args) throws InterruptedException {
                new Thread(new Runnable() {
                        @Override
                        public void run() {
                                int i = 0;
                                while (!cancel) {
                                        i++;
                                }
                                System.out.println("Done!");
                        }

                }).start();

                System.out.println("OS: " + System.getProperty("os.name") + " " + System.getProperty("os.arch") + "\n"
                                + "JavaVersion: " + System.getProperty("java.version") + "\n");
                Thread.sleep(2048);
                cancel = true;
                System.out.println("flag cancel set to true");
        }
}
```

笔者只有一台古老的机器（Intel(R) Core(TM)2 Quad CPU Q6600 @ 2.40GHz），执行结果如下：

``` plain
yuan@yuan-HP:~/learn$ jdk1.7.0_25/bin/java MemoryBarrierDemo
OS: Linux amd64
JavaVersion: 1.7.0_25

flag cancel set to true
```

程序一直处于循环递增`i`的状态，没有退出。也就是说程序中启动的子线程“看不到”主线程对共享变量`cancel`的改动，这就是从高级语言层面所看到的memory barrier，想要让子线程“看到”改变，那么就得让改变“穿过”memory barrier。

如何做？在_Java Concurrency in Practice_的3.1节介绍了两种方法——locking和volatile variable。在此我们试着用volatile关键字，将前面程序中的`private static boolean cancel;`改为`private static volatile boolean cancel;`。再次编译执行程序，结果如下：

``` plain
yuan@yuan-HP:~/learn$ jdk1.7.0_25/bin/java MemoryBarrierDemo
OS: Linux amd64
JavaVersion: 1.7.0_25

flag cancel set to true
Done!
yuan@yuan-HP:~/learn$
```

如此程序就可以按照设想的那像正常退出了，很简单吧。

好的，下面就开始探索笔者困惑的部分了。

## 汇编代码

下面我们通过添加几个参数打印出反汇编的代码看看。

没有volatile的版本：

``` plain
yuan@yuan-HP:~/learn$ jdk1.7.0_25/bin/java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly MemoryBarrierDemo
Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output
OS: Linux amd64
JavaVersion: 1.7.0_25

Loaded disassembler from /home/yuan/learn/jdk1.7.0_25/jre/lib/amd64/hsdis-amd64.so
Decoding compiled method 0x00007fbe883df890:
Code:
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} 'access$000' '()Z' in 'MemoryBarrierDemo'
  #           [sp+0x20]  (sp of caller)
  0x00007fbe883df9c0: sub    $0x18,%rsp
  0x00007fbe883df9c7: mov    %rbp,0x10(%rsp)    ;*synchronization entry
                                                ; - MemoryBarrierDemo::access$000@-1 (line 1)
  0x00007fbe883df9cc: mov    $0xebcfc590,%r10   ;   {oop(a 'java/lang/Class' = 'MemoryBarrierDemo')}
  0x00007fbe883df9d6: movzbl 0x70(%r10),%eax    ;*getstatic cancel
                                                ; - MemoryBarrierDemo::access$000@0 (line 1)
  0x00007fbe883df9db: add    $0x10,%rsp
  0x00007fbe883df9df: pop    %rbp
  0x00007fbe883df9e0: test   %eax,0x9c6b61a(%rip)        # 0x00007fbe9204b000
                                                ;   {poll_return}
  0x00007fbe883df9e6: retq
  0x00007fbe883df9e7: hlt
  0x00007fbe883df9e8: hlt
  0x00007fbe883df9e9: hlt
  0x00007fbe883df9ea: hlt
  0x00007fbe883df9eb: hlt
  0x00007fbe883df9ec: hlt
  0x00007fbe883df9ed: hlt
  0x00007fbe883df9ee: hlt
  0x00007fbe883df9ef: hlt
  0x00007fbe883df9f0: hlt
  0x00007fbe883df9f1: hlt
  0x00007fbe883df9f2: hlt
  0x00007fbe883df9f3: hlt
  0x00007fbe883df9f4: hlt
  0x00007fbe883df9f5: hlt
  0x00007fbe883df9f6: hlt
  0x00007fbe883df9f7: hlt
  0x00007fbe883df9f8: hlt
  0x00007fbe883df9f9: hlt
  0x00007fbe883df9fa: hlt
  0x00007fbe883df9fb: hlt
  0x00007fbe883df9fc: hlt
  0x00007fbe883df9fd: hlt
  0x00007fbe883df9fe: hlt
  0x00007fbe883df9ff: hlt
[Exception Handler]
[Stub Code]
  0x00007fbe883dfa00: jmpq   0x00007fbe883dc860  ;   {no_reloc}
[Deopt Handler Code]
  0x00007fbe883dfa05: callq  0x00007fbe883dfa0a
  0x00007fbe883dfa0a: subq   $0x5,(%rsp)
  0x00007fbe883dfa0f: jmpq   0x00007fbe883b6c00  ;   {runtime_call}
  0x00007fbe883dfa14: hlt
  0x00007fbe883dfa15: hlt
  0x00007fbe883dfa16: hlt
  0x00007fbe883dfa17: hlt
Decoding compiled method 0x00007fbe883ddcd0:
Code:
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} 'run' '()V' in 'MemoryBarrierDemo$1'
  0x00007fbe883dde20: callq  0x00007fbe90e52370  ;   {runtime_call}
  0x00007fbe883dde25: nopw   0x0(%rax,%rax,1)
  0x00007fbe883dde30: mov    %eax,-0x14000(%rsp)
  0x00007fbe883dde37: push   %rbp
  0x00007fbe883dde38: sub    $0x10,%rsp
  0x00007fbe883dde3c: mov    (%rsi),%ebx
  0x00007fbe883dde3e: mov    %rsi,%rdi
  0x00007fbe883dde41: mov    $0x7fbe90edf400,%r10
  0x00007fbe883dde4b: callq  *%r10              ;*invokestatic access$000
                                                ; - MemoryBarrierDemo$1::run@2 (line 10)
  0x00007fbe883dde4e: mov    $0xebcfc590,%r10   ;   {oop(a 'java/lang/Class' = 'MemoryBarrierDemo')}
  0x00007fbe883dde58: movzbl 0x70(%r10),%r11d
  0x00007fbe883dde5d: test   %r11d,%r11d
  0x00007fbe883dde60: jne    0x00007fbe883dde6c  ;*goto
                                                ; - MemoryBarrierDemo$1::run@11 (line 11)
  0x00007fbe883dde62: inc    %ebx               ; OopMap{off=68}
                                                ;*goto
                                                ; - MemoryBarrierDemo$1::run@11 (line 11)
  0x00007fbe883dde64: test   %eax,0x9c6d196(%rip)        # 0x00007fbe9204b000
                                                ;*goto
                                                ; - MemoryBarrierDemo$1::run@11 (line 11)
                                                ;   {poll}
  0x00007fbe883dde6a: jmp    0x00007fbe883dde62
  0x00007fbe883dde6c: mov    $0xebcb0d10,%r10   ;   {oop(a 'java/lang/Class' = 'java/lang/System')}
  0x00007fbe883dde76: mov    0x74(%r10),%r11d   ;*getstatic out
                                                ; - MemoryBarrierDemo$1::run@14 (line 13)
  0x00007fbe883dde7a: test   %r11d,%r11d
  0x00007fbe883dde7d: je     0x00007fbe883ddea0  ;*ifne
                                                ; - MemoryBarrierDemo$1::run@5 (line 10)
  0x00007fbe883dde7f: mov    %r11,%rsi          ;*getstatic out
                                                ; - MemoryBarrierDemo$1::run@14 (line 13)
  0x00007fbe883dde82: mov    $0xebd4b960,%rdx   ;   {oop("Done!")}
  0x00007fbe883dde8c: xchg   %ax,%ax
  0x00007fbe883dde8f: callq  0x00007fbe883b5c60  ; OopMap{off=116}
                                                ;*invokevirtual println
                                                ; - MemoryBarrierDemo$1::run@19 (line 13)
                                                ;   {optimized virtual_call}
  0x00007fbe883dde94: add    $0x10,%rsp
  0x00007fbe883dde98: pop    %rbp
  0x00007fbe883dde99: test   %eax,0x9c6d161(%rip)        # 0x00007fbe9204b000
                                                ;   {poll_return}
  0x00007fbe883dde9f: retq
  0x00007fbe883ddea0: mov    $0xfffffff6,%esi
  0x00007fbe883ddea5: xchg   %ax,%ax
  0x00007fbe883ddea7: callq  0x00007fbe883b7020  ; OopMap{off=140}
                                                ;*invokevirtual println
                                                ; - MemoryBarrierDemo$1::run@19 (line 13)
                                                ;   {runtime_call}
  0x00007fbe883ddeac: callq  0x00007fbe90e52370  ;*invokevirtual println
                                                ; - MemoryBarrierDemo$1::run@19 (line 13)
                                                ;   {runtime_call}
  0x00007fbe883ddeb1: mov    %rax,%rsi
  0x00007fbe883ddeb4: add    $0x10,%rsp
  0x00007fbe883ddeb8: pop    %rbp
  0x00007fbe883ddeb9: jmpq   0x00007fbe883df5e0  ;*iinc
                                                ; - MemoryBarrierDemo$1::run@8 (line 11)
                                                ;   {runtime_call}
  0x00007fbe883ddebe: callq  0x00007fbe90e52370  ;   {runtime_call}
  0x00007fbe883ddec3: hlt
  0x00007fbe883ddec4: hlt
  0x00007fbe883ddec5: hlt
  0x00007fbe883ddec6: hlt
  0x00007fbe883ddec7: hlt
  0x00007fbe883ddec8: hlt
  0x00007fbe883ddec9: hlt
  0x00007fbe883ddeca: hlt
  0x00007fbe883ddecb: hlt
  0x00007fbe883ddecc: hlt
  0x00007fbe883ddecd: hlt
  0x00007fbe883ddece: hlt
  0x00007fbe883ddecf: hlt
  0x00007fbe883dded0: hlt
  0x00007fbe883dded1: hlt
  0x00007fbe883dded2: hlt
  0x00007fbe883dded3: hlt
  0x00007fbe883dded4: hlt
  0x00007fbe883dded5: hlt
  0x00007fbe883dded6: hlt
  0x00007fbe883dded7: hlt
  0x00007fbe883dded8: hlt
  0x00007fbe883dded9: hlt
  0x00007fbe883ddeda: hlt
  0x00007fbe883ddedb: hlt
  0x00007fbe883ddedc: hlt
  0x00007fbe883ddedd: hlt
  0x00007fbe883ddede: hlt
  0x00007fbe883ddedf: hlt
[Stub Code]
  0x00007fbe883ddee0: mov    $0x0,%rbx          ;   {no_reloc}
  0x00007fbe883ddeea: jmpq   0x00007fbe883ddeea  ;   {runtime_call}
[Exception Handler]
  0x00007fbe883ddeef: jmpq   0x00007fbe883dc860  ;   {runtime_call}
[Deopt Handler Code]
  0x00007fbe883ddef4: callq  0x00007fbe883ddef9
  0x00007fbe883ddef9: subq   $0x5,(%rsp)
  0x00007fbe883ddefe: jmpq   0x00007fbe883b6c00  ;   {runtime_call}
  0x00007fbe883ddf03: hlt
  0x00007fbe883ddf04: hlt
  0x00007fbe883ddf05: hlt
  0x00007fbe883ddf06: hlt
  0x00007fbe883ddf07: hlt
flag cancel set to true
```

带有volatile的版本：

``` plain
yuan@yuan-HP:~/learn$ jdk1.7.0_25/bin/java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly MemoryBarrierDemo
Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output
OS: Linux amd64
JavaVersion: 1.7.0_25

Loaded disassembler from /home/yuan/learn/jdk1.7.0_25/jre/lib/amd64/hsdis-amd64.so
Decoding compiled method 0x00007fb865061890:
Code:
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} 'access$000' '()Z' in 'MemoryBarrierDemo'
  #           [sp+0x20]  (sp of caller)
  0x00007fb8650619c0: sub    $0x18,%rsp
  0x00007fb8650619c7: mov    %rbp,0x10(%rsp)    ;*synchronization entry
                                                ; - MemoryBarrierDemo::access$000@-1 (line 1)
  0x00007fb8650619cc: mov    $0xebcfc590,%r10   ;   {oop(a 'java/lang/Class' = 'MemoryBarrierDemo')}
  0x00007fb8650619d6: movzbl 0x70(%r10),%eax    ;*getstatic cancel
                                                ; - MemoryBarrierDemo::access$000@0 (line 1)
  0x00007fb8650619db: add    $0x10,%rsp
  0x00007fb8650619df: pop    %rbp
  0x00007fb8650619e0: test   %eax,0xb55e61a(%rip)        # 0x00007fb8705c0000
                                                ;   {poll_return}
  0x00007fb8650619e6: retq
  0x00007fb8650619e7: hlt
  0x00007fb8650619e8: hlt
  0x00007fb8650619e9: hlt
  0x00007fb8650619ea: hlt
  0x00007fb8650619eb: hlt
  0x00007fb8650619ec: hlt
  0x00007fb8650619ed: hlt
  0x00007fb8650619ee: hlt
  0x00007fb8650619ef: hlt
  0x00007fb8650619f0: hlt
  0x00007fb8650619f1: hlt
  0x00007fb8650619f2: hlt
  0x00007fb8650619f3: hlt
  0x00007fb8650619f4: hlt
  0x00007fb8650619f5: hlt
  0x00007fb8650619f6: hlt
  0x00007fb8650619f7: hlt
  0x00007fb8650619f8: hlt
  0x00007fb8650619f9: hlt
  0x00007fb8650619fa: hlt
  0x00007fb8650619fb: hlt
  0x00007fb8650619fc: hlt
  0x00007fb8650619fd: hlt
  0x00007fb8650619fe: hlt
  0x00007fb8650619ff: hlt
[Exception Handler]
[Stub Code]
  0x00007fb865061a00: jmpq   0x00007fb86505e860  ;   {no_reloc}
[Deopt Handler Code]
  0x00007fb865061a05: callq  0x00007fb865061a0a
  0x00007fb865061a0a: subq   $0x5,(%rsp)
  0x00007fb865061a0f: jmpq   0x00007fb865038c00  ;   {runtime_call}
  0x00007fb865061a14: hlt
  0x00007fb865061a15: hlt
  0x00007fb865061a16: hlt
  0x00007fb865061a17: hlt
Decoding compiled method 0x00007fb86505fc90:
Code:
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} 'run' '()V' in 'MemoryBarrierDemo$1'
  0x00007fb86505fde0: callq  0x00007fb86f3c7370  ;   {runtime_call}
  0x00007fb86505fde5: nopw   0x0(%rax,%rax,1)
  0x00007fb86505fdf0: mov    %eax,-0x14000(%rsp)
  0x00007fb86505fdf7: push   %rbp
  0x00007fb86505fdf8: sub    $0x10,%rsp
  0x00007fb86505fdfc: mov    (%rsi),%ebp
  0x00007fb86505fdfe: mov    %rsi,%rdi
  0x00007fb86505fe01: mov    $0x7fb86f454400,%r10
  0x00007fb86505fe0b: callq  *%r10              ;*synchronization entry
                                                ; - MemoryBarrierDemo::access$000@-1 (line 1)
                                                ; - MemoryBarrierDemo$1::run@2 (line 10)
  0x00007fb86505fe0e: mov    $0xebcfc590,%r10   ;   {oop(a 'java/lang/Class' = 'MemoryBarrierDemo')}
  0x00007fb86505fe18: movzbl 0x70(%r10),%r8d    ;*getstatic cancel
                                                ; - MemoryBarrierDemo::access$000@0 (line 1)
                                                ; - MemoryBarrierDemo$1::run@2 (line 10)
  0x00007fb86505fe1d: test   %r8d,%r8d
  0x00007fb86505fe20: jne    0x00007fb86505fe42  ;*ifne
                                                ; - MemoryBarrierDemo$1::run@5 (line 10)
  0x00007fb86505fe22: nopw   0x0(%rax,%rax,1)
  0x00007fb86505fe2c: xchg   %ax,%ax            ;*iinc
                                                ; - MemoryBarrierDemo$1::run@8 (line 11)
  0x00007fb86505fe30: movzbl 0x70(%r10),%r11d   ;*getstatic cancel
                                                ; - MemoryBarrierDemo::access$000@0 (line 1)
                                                ; - MemoryBarrierDemo$1::run@2 (line 10)
  0x00007fb86505fe35: inc    %ebp               ; OopMap{r10=Oop off=87}
                                                ;*goto
                                                ; - MemoryBarrierDemo$1::run@11 (line 11)
  0x00007fb86505fe37: test   %eax,0xb5601c3(%rip)        # 0x00007fb8705c0000
                                                ;*getstatic cancel
                                                ; - MemoryBarrierDemo::access$000@0 (line 1)
                                                ; - MemoryBarrierDemo$1::run@2 (line 10)
                                                ;   {poll}
  0x00007fb86505fe3d: test   %r11d,%r11d
  0x00007fb86505fe40: je     0x00007fb86505fe30  ;*ifne
                                                ; - MemoryBarrierDemo$1::run@5 (line 10)
  0x00007fb86505fe42: mov    $0xebcb0d10,%r10   ;   {oop(a 'java/lang/Class' = 'java/lang/System')}
  0x00007fb86505fe4c: mov    0x74(%r10),%r10d   ;*getstatic out
                                                ; - MemoryBarrierDemo$1::run@14 (line 13)
  0x00007fb86505fe50: test   %r10d,%r10d
  0x00007fb86505fe53: je     0x00007fb86505fe74  ;*invokevirtual println
                                                ; - MemoryBarrierDemo$1::run@19 (line 13)
  0x00007fb86505fe55: mov    %r10,%rsi          ;*getstatic out
                                                ; - MemoryBarrierDemo$1::run@14 (line 13)
  0x00007fb86505fe58: mov    $0xebd4b960,%rdx   ;   {oop("Done!")}
  0x00007fb86505fe62: nop
  0x00007fb86505fe63: callq  0x00007fb865037c60  ; OopMap{off=136}
                                                ;*invokevirtual println
                                                ; - MemoryBarrierDemo$1::run@19 (line 13)
                                                ;   {optimized virtual_call}
  0x00007fb86505fe68: add    $0x10,%rsp
  0x00007fb86505fe6c: pop    %rbp
  0x00007fb86505fe6d: test   %eax,0xb56018d(%rip)        # 0x00007fb8705c0000
                                                ;   {poll_return}
  0x00007fb86505fe73: retq
  0x00007fb86505fe74: mov    $0xfffffff6,%esi
  0x00007fb86505fe79: xchg   %ax,%ax
  0x00007fb86505fe7b: callq  0x00007fb865039020  ; OopMap{off=160}
                                                ;*invokevirtual println
                                                ; - MemoryBarrierDemo$1::run@19 (line 13)
                                                ;   {runtime_call}
  0x00007fb86505fe80: callq  0x00007fb86f3c7370  ;*invokevirtual println
                                                ; - MemoryBarrierDemo$1::run@19 (line 13)
                                                ;   {runtime_call}
  0x00007fb86505fe85: mov    %rax,%rsi
  0x00007fb86505fe88: add    $0x10,%rsp
  0x00007fb86505fe8c: pop    %rbp
  0x00007fb86505fe8d: jmpq   0x00007fb8650615e0  ;   {runtime_call}
  0x00007fb86505fe92: hlt
  0x00007fb86505fe93: hlt
  0x00007fb86505fe94: hlt
  0x00007fb86505fe95: hlt
  0x00007fb86505fe96: hlt
  0x00007fb86505fe97: hlt
  0x00007fb86505fe98: hlt
  0x00007fb86505fe99: hlt
  0x00007fb86505fe9a: hlt
  0x00007fb86505fe9b: hlt
  0x00007fb86505fe9c: hlt
  0x00007fb86505fe9d: hlt
  0x00007fb86505fe9e: hlt
  0x00007fb86505fe9f: hlt
[Stub Code]
  0x00007fb86505fea0: mov    $0x0,%rbx          ;   {no_reloc}
  0x00007fb86505feaa: jmpq   0x00007fb86505feaa  ;   {runtime_call}
[Exception Handler]
  0x00007fb86505feaf: jmpq   0x00007fb86505e860  ;   {runtime_call}
[Deopt Handler Code]
  0x00007fb86505feb4: callq  0x00007fb86505feb9
  0x00007fb86505feb9: subq   $0x5,(%rsp)
  0x00007fb86505febe: jmpq   0x00007fb865038c00  ;   {runtime_call}
  0x00007fb86505fec3: hlt
  0x00007fb86505fec4: hlt
  0x00007fb86505fec5: hlt
  0x00007fb86505fec6: hlt
  0x00007fb86505fec7: hlt
flag cancel set to true
Done!
yuan@yuan-HP:~/learn$
```

现在我们分析一下。在没有volatile的版本中，循环递增部分的汇编代码很有趣，

``` plain
  0x00007fbe883dde58: movzbl 0x70(%r10),%r11d
  0x00007fbe883dde5d: test   %r11d,%r11d
  0x00007fbe883dde60: jne    0x00007fbe883dde6c  ;*goto
                                                ; - MemoryBarrierDemo$1::run@11 (line 11)
  0x00007fbe883dde62: inc    %ebx               ; OopMap{off=68}
                                                ;*goto
                                                ; - MemoryBarrierDemo$1::run@11 (line 11)
  0x00007fbe883dde64: test   %eax,0x9c6d196(%rip)        # 0x00007fbe9204b000
                                                ;*goto
                                                ; - MemoryBarrierDemo$1::run@11 (line 11)
                                                ;   {poll}
  0x00007fbe883dde6a: jmp    0x00007fbe883dde62
```

进入循环时检查过一次`cancel`，但是在后面却成了`jmp`无条件跳转，可想而知这肯定是编译器的“功劳”。在此笔者就先不深究这里所做的编译优化的原因了，只是简单的相信编译器已经获得了足够多的信息来判断`cancel`值在该线程是不变的。反过来推理，编译器的开发者肯定是对硬件层面的memory barrier有深入的理解才会做这个优化，这样也是本文想要探索的。

在带有volatile的版本中，循环递增部分的汇编代码如下：

``` plain
  0x00007fb86505fe18: movzbl 0x70(%r10),%r8d    ;*getstatic cancel
                                                ; - MemoryBarrierDemo::access$000@0 (line 1)
                                                ; - MemoryBarrierDemo$1::run@2 (line 10)
  0x00007fb86505fe1d: test   %r8d,%r8d
  0x00007fb86505fe20: jne    0x00007fb86505fe42  ;*ifne
                                                ; - MemoryBarrierDemo$1::run@5 (line 10)
  0x00007fb86505fe22: nopw   0x0(%rax,%rax,1)
  0x00007fb86505fe2c: xchg   %ax,%ax            ;*iinc
                                                ; - MemoryBarrierDemo$1::run@8 (line 11)
  0x00007fb86505fe30: movzbl 0x70(%r10),%r11d   ;*getstatic cancel
                                                ; - MemoryBarrierDemo::access$000@0 (line 1)
                                                ; - MemoryBarrierDemo$1::run@2 (line 10)
  0x00007fb86505fe35: inc    %ebp               ; OopMap{r10=Oop off=87}
                                                ;*goto
                                                ; - MemoryBarrierDemo$1::run@11 (line 11)
  0x00007fb86505fe37: test   %eax,0xb5601c3(%rip)        # 0x00007fb8705c0000
                                                ;*getstatic cancel
                                                ; - MemoryBarrierDemo::access$000@0 (line 1)
                                                ; - MemoryBarrierDemo$1::run@2 (line 10)
                                                ;   {poll}
  0x00007fb86505fe3d: test   %r11d,%r11d
  0x00007fb86505fe40: je     0x00007fb86505fe30  ;*ifne
                                                ; - MemoryBarrierDemo$1::run@5 (line 10)
  0x00007fb86505fe42: mov    $0xebcb0d10,%r10   ;   {oop(a 'java/lang/Class' = 'java/lang/System')}
```

该版本的主要不同是在于获取cacel前执行了`xchg   %ax,%ax`。

xchg是什么东东？查_Intel 64 and IA-32 Architectures Software Developer's Manual Combined Volumes 3A, 3B, and 3C: System Programming Guide_：

8-3页：

> 8.1.2 Bus Locking

> Intel 64 and IA-32 processors provide a LOCK# signal that is asserted automatically during certain critical memory operations to lock the system bus or equivalent link. While this output signal is asserted, requests from other processors or bus agents for control of the bus are blocked. Software can specify other occasions when the LOCK semantics are to be followed by prepending the LOCK prefix to an instruction.

> In the case of the Intel386, Intel486, and Pentium processors , explicitly locked instructions will result in the assertion of the LOCK# signal. It is the responsibility of the hardware designer to make the LOCK# signal available in system hardware to control memory accesses among processors.

> For the P6 and more recent processor families, if the memory area being accessed is cached internally in the processor, the LOCK# signal is generally not asserted; instead, locking is only applied to the processor’s caches (see Section 8.1.4, “Effects of a LOCK Operation on Internal Processor Caches”).

> 8.1.2.1   Automatic Locking

> The operations on which the processor automatically follows the LOCK semantics are as follows:

> • When executing an XCHG instruction that references memory.

8-15页：

> Synchronization mechanisms in multiple-processor systems may depend upon a strong memory-ordering model. Here, a program can use a locking instruction such as the XCHG instruction or the LOCK  prefix to ensure that a read-modify-write operation on memory is carried out atomically. Locking operations typically operate like I/O operations in that they wait for all previous instructions  to complete and for all buffered writes to drain to memory (see Section 8.1.2, “Bus Locking”).

8-16页：

> Intel recommends that software written to run on Intel Core 2 Duo, Intel Atom, Intel Core Duo, Pentium 4, Intel Xeon, and P6 family processors assume the processor-ordering model or a weaker memory-ordering model. The Intel Core 2 Duo, Intel Atom, Intel Core Duo, Pentium 4, Intel Xeon, and P6 family processors do not implement a strong memory-ordering model, except when using the UC memory type. Despite the fact that Pentium 4, Intel Xeon, and P6 family processors support processor ordering, Intel does not guarantee that future processors will support this model. To make software portable to futu re processors, it is recommended that operating systems provide critical region and resource control constructs and API’s (application program interfaces) based on I/O, locking, and/or serializing instructions be used to synchronize access to shared areas of memory in multiple-processor systems. Also, software should not depend on processor ordering in situations where the system hardware does not support this memory-ordering model.

读完之后，笔者大致了解了三点：

1. 执行XCHG指令会自动跟随LOCK语义；

2. locking操作会等待前面所有的指令完成，并且将所有buffered writes写到内存；

3. memory-ordering model的强度是变化的，软件不能依赖processor ordering。这就解释了第一个版本中的编译优化，编译器从高级语言中得不到显式的信息（比如Locking和volatile），所以就默认使用processor-ordering model，编译器就会尽可能的优化性能；而在第二个版本中，有显式的volatile关键字，所以编译器就会加入XCHG指令，以实现strong memory-ordering model。

等一下，好像哪里有问题。就算`xchg`触发了`LOCK`语义，但是缓存中的`cancel`并没有改变，所以也不会触发cache coherency protocol；就算`cancel`所在的cache line由于别的原因被修改了，而此前主线程没有对`cancel`进行修改的话，也是没有用呀。再从头开始想想这个逻辑，如果让笔者设计编译器，笔者应该会在`cancel = true;`这行之后加上LOCK语义。再回过头看看汇编代码，竟然找不到对应的代码！

经过笔者多次尝试，最后通过加入`-Xcomp`打印出了所需的信息，另外对Java代码也做了点儿改动。（笔者对此有蛮多疑惑，暂且存疑吧-_-）

修改后的代码如下：

```java
public class MemoryBarrierDemo {

        private static volatile boolean cancel = false;
        private static void setCancel() {
                cancel = true;
        }

        public static void main(String[] args) throws InterruptedException {
                new Thread(new Runnable() {
                        @Override
                        public void run() {
                                int i = 0;
                                while (!cancel) {
                                        i++;
                                }
                                System.out.println("Done!");
                        }

                }).start();

                System.out.println("OS: " + System.getProperty("os.name") + " " + System.getProperty("os.arch") + "\n"
                                + "JavaVersion: " + System.getProperty("java.version") + "\n");
                Thread.sleep(2048);
                //cancel = true;
                setCancel();
                System.out.println("flag cancel set to true");
        }
}
```

汇编代码如下：

``` plain
yuan@yuan-HP:~/learn$ jdk1.7.0_25/bin/java -Xcomp  -XX:+UnlockDiagnosticVMOptions -XX:PrintAssemblyOptions=hsdis-print-bytes -XX:CompileCommand=print,MemoryBarrierDemo.* MemoryBarrierDemo
CompilerOracle: print MemoryBarrierDemo.*
Java HotSpot(TM) 64-Bit Server VM warning: printing of assembly code is enabled; turning on DebugNonSafepoints to gain additional output
Compiled method (c2)     888  370             MemoryBarrierDemo::<clinit> (5 bytes)
 total in heap  [0x00007fbaec8ae2d0,0x00007fbaec8ae4a0] = 464
 relocation     [0x00007fbaec8ae3f0,0x00007fbaec8ae400] = 16
 main code      [0x00007fbaec8ae400,0x00007fbaec8ae440] = 64
 stub code      [0x00007fbaec8ae440,0x00007fbaec8ae458] = 24
 oops           [0x00007fbaec8ae458,0x00007fbaec8ae460] = 8
 scopes data    [0x00007fbaec8ae460,0x00007fbaec8ae468] = 8
 scopes pcs     [0x00007fbaec8ae468,0x00007fbaec8ae498] = 48
 dependencies   [0x00007fbaec8ae498,0x00007fbaec8ae4a0] = 8
Loaded disassembler from /home/yuan/learn/jdk1.7.0_25/jre/lib/amd64/hsdis-amd64.so
Decoding compiled method 0x00007fbaec8ae2d0:
Code:
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} '<clinit>' '()V' in 'MemoryBarrierDemo'
  #           [sp+0x20]  (sp of caller)
  0x00007fbaec8ae400: sub    $0x18,%rsp         ;...4881ec18 000000
  0x00007fbaec8ae407: mov    %rbp,0x10(%rsp)    ;...48896c24 10
  0x00007fbaec8ae40c: mov    $0xebd9deb8,%r10   ;...49bab8de d9eb00
                                                ;...000000
                                                ;   {oop(a 'java/lang/Class' = 'MemoryBarrierDemo')}
  0x00007fbaec8ae416: mov    %r12b,0x70(%r10)   ;...45886270
  0x00007fbaec8ae41a: lock addl $0x0,(%rsp)     ;...f0830424 00
                                                ;*putstatic cancel
                                                ; - MemoryBarrierDemo::<clinit>@1 (line 3)
  0x00007fbaec8ae41f: add    $0x10,%rsp         ;...4883c410
  0x00007fbaec8ae423: pop    %rbp               ;...5d
  0x00007fbaec8ae424: test   %eax,0x9c59bd6(%rip)        # 0x00007fbaf6508000
                                                ;...8505d69b c509
                                                ;   {poll_return}
  0x00007fbaec8ae42a: retq                      ;...c3
  0x00007fbaec8ae42b: hlt                       ;...f4
  0x00007fbaec8ae42c: hlt                       ;...f4
  0x00007fbaec8ae42d: hlt                       ;...f4
  0x00007fbaec8ae42e: hlt                       ;...f4
  0x00007fbaec8ae42f: hlt                       ;...f4
  0x00007fbaec8ae430: hlt                       ;...f4
  0x00007fbaec8ae431: hlt                       ;...f4
  0x00007fbaec8ae432: hlt                       ;...f4
  0x00007fbaec8ae433: hlt                       ;...f4
  0x00007fbaec8ae434: hlt                       ;...f4
  0x00007fbaec8ae435: hlt                       ;...f4
  0x00007fbaec8ae436: hlt                       ;...f4
  0x00007fbaec8ae437: hlt                       ;...f4
  0x00007fbaec8ae438: hlt                       ;...f4
  0x00007fbaec8ae439: hlt                       ;...f4
  0x00007fbaec8ae43a: hlt                       ;...f4
  0x00007fbaec8ae43b: hlt                       ;...f4
  0x00007fbaec8ae43c: hlt                       ;...f4
  0x00007fbaec8ae43d: hlt                       ;...f4
  0x00007fbaec8ae43e: hlt                       ;...f4
  0x00007fbaec8ae43f: hlt                       ;...f4
[Exception Handler]
[Stub Code]
  0x00007fbaec8ae440: jmpq   0x00007fbaec80f860  ;...e91b14f6 ff
                                                ;   {no_reloc}
[Deopt Handler Code]
  0x00007fbaec8ae445: callq  0x00007fbaec8ae44a  ;...e8000000 00
  0x00007fbaec8ae44a: subq   $0x5,(%rsp)        ;...48832c24 05
  0x00007fbaec8ae44f: jmpq   0x00007fbaec7eac00  ;...e9acc7f3 ff
                                                ;   {runtime_call}
  0x00007fbaec8ae454: hlt                       ;...f4
  0x00007fbaec8ae455: hlt                       ;...f4
  0x00007fbaec8ae456: hlt                       ;...f4
  0x00007fbaec8ae457: hlt                       ;...f4
OopMapSet contains 0 OopMaps

Compiled method (c2)     893  371             MemoryBarrierDemo::main (100 bytes)
 total in heap  [0x00007fbaec8b10d0,0x00007fbaec8b12b0] = 480
 relocation     [0x00007fbaec8b11f0,0x00007fbaec8b1200] = 16
 main code      [0x00007fbaec8b1200,0x00007fbaec8b1220] = 32
 stub code      [0x00007fbaec8b1220,0x00007fbaec8b1238] = 24
 oops           [0x00007fbaec8b1238,0x00007fbaec8b1240] = 8
 scopes data    [0x00007fbaec8b1240,0x00007fbaec8b1258] = 24
 scopes pcs     [0x00007fbaec8b1258,0x00007fbaec8b12a8] = 80
 dependencies   [0x00007fbaec8b12a8,0x00007fbaec8b12b0] = 8
Decoding compiled method 0x00007fbaec8b10d0:
Code:
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} 'main' '([Ljava/lang/String;)V' in 'MemoryBarrierDemo'
  # parm0:    rsi:rsi   = '[Ljava/lang/String;'
  #           [sp+0x20]  (sp of caller)
  0x00007fbaec8b1200: mov    %eax,-0x14000(%rsp)  ;...89842400 c0feff
  0x00007fbaec8b1207: push   %rbp               ;...55
  0x00007fbaec8b1208: sub    $0x10,%rsp         ;...4883ec10
                                                ;*synchronization entry
                                                ; - MemoryBarrierDemo::main@-1 (line 9)
  0x00007fbaec8b120c: mov    $0x3,%esi          ;...be030000 00
  0x00007fbaec8b1211: xchg   %ax,%ax            ;...6690
  0x00007fbaec8b1213: callq  0x00007fbaec7eb020  ;...e8089ef3 ff
                                                ; OopMap{off=24}
                                                ;*new  ; - MemoryBarrierDemo::main@0 (line 9)
                                                ;   {runtime_call}
  0x00007fbaec8b1218: callq  0x00007fbaf530f370  ;...e853e1a5 08
                                                ;*new  ; - MemoryBarrierDemo::main@0 (line 9)
                                                ;   {runtime_call}
  0x00007fbaec8b121d: hlt                       ;...f4
  0x00007fbaec8b121e: hlt                       ;...f4
  0x00007fbaec8b121f: hlt                       ;...f4
[Exception Handler]
[Stub Code]
  0x00007fbaec8b1220: jmpq   0x00007fbaec80f860  ;...e93be6f5 ff
                                                ;   {no_reloc}
[Deopt Handler Code]
  0x00007fbaec8b1225: callq  0x00007fbaec8b122a  ;...e8000000 00
  0x00007fbaec8b122a: subq   $0x5,(%rsp)        ;...48832c24 05
  0x00007fbaec8b122f: jmpq   0x00007fbaec7eac00  ;...e9cc99f3 ff
                                                ;   {runtime_call}
  0x00007fbaec8b1234: hlt                       ;...f4
  0x00007fbaec8b1235: hlt                       ;...f4
  0x00007fbaec8b1236: hlt                       ;...f4
  0x00007fbaec8b1237: hlt                       ;...f4
OopMapSet contains 1 OopMaps

#0
OopMap{off=24}
Compiled method (c2)     994  398             MemoryBarrierDemo::access$000 (4 bytes)
 total in heap  [0x00007fbaec8b9390,0x00007fbaec8b9578] = 488
 relocation     [0x00007fbaec8b94b0,0x00007fbaec8b94c0] = 16
 main code      [0x00007fbaec8b94c0,0x00007fbaec8b9500] = 64
 stub code      [0x00007fbaec8b9500,0x00007fbaec8b9518] = 24
 oops           [0x00007fbaec8b9518,0x00007fbaec8b9520] = 8
 scopes data    [0x00007fbaec8b9520,0x00007fbaec8b9530] = 16
 scopes pcs     [0x00007fbaec8b9530,0x00007fbaec8b9570] = 64
 dependencies   [0x00007fbaec8b9570,0x00007fbaec8b9578] = 8
Decoding compiled method 0x00007fbaec8b9390:
Code:
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} 'access$000' '()Z' in 'MemoryBarrierDemo'
  #           [sp+0x20]  (sp of caller)
  0x00007fbaec8b94c0: sub    $0x18,%rsp         ;...4881ec18 000000
  0x00007fbaec8b94c7: mov    %rbp,0x10(%rsp)    ;...48896c24 10
                                                ;*synchronization entry
                                                ; - MemoryBarrierDemo::access$000@-1 (line 1)
  0x00007fbaec8b94cc: mov    $0xebd9deb8,%r10   ;...49bab8de d9eb00
                                                ;...000000
                                                ;   {oop(a 'java/lang/Class' = 'MemoryBarrierDemo')}
  0x00007fbaec8b94d6: movzbl 0x70(%r10),%eax    ;...410fb642 70
                                                ;*getstatic cancel
                                                ; - MemoryBarrierDemo::access$000@0 (line 1)
  0x00007fbaec8b94db: add    $0x10,%rsp         ;...4883c410
  0x00007fbaec8b94df: pop    %rbp               ;...5d
  0x00007fbaec8b94e0: test   %eax,0x9c4eb1a(%rip)        # 0x00007fbaf6508000
                                                ;...85051aeb c409
                                                ;   {poll_return}
  0x00007fbaec8b94e6: retq                      ;...c3
  0x00007fbaec8b94e7: hlt                       ;...f4
  0x00007fbaec8b94e8: hlt                       ;...f4
  0x00007fbaec8b94e9: hlt                       ;...f4
  0x00007fbaec8b94ea: hlt                       ;...f4
  0x00007fbaec8b94eb: hlt                       ;...f4
  0x00007fbaec8b94ec: hlt                       ;...f4
  0x00007fbaec8b94ed: hlt                       ;...f4
  0x00007fbaec8b94ee: hlt                       ;...f4
  0x00007fbaec8b94ef: hlt                       ;...f4
  0x00007fbaec8b94f0: hlt                       ;...f4
  0x00007fbaec8b94f1: hlt                       ;...f4
  0x00007fbaec8b94f2: hlt                       ;...f4
  0x00007fbaec8b94f3: hlt                       ;...f4
  0x00007fbaec8b94f4: hlt                       ;...f4
  0x00007fbaec8b94f5: hlt                       ;...f4
  0x00007fbaec8b94f6: hlt                       ;...f4
  0x00007fbaec8b94f7: hlt                       ;...f4
  0x00007fbaec8b94f8: hlt                       ;...f4
  0x00007fbaec8b94f9: hlt                       ;...f4
  0x00007fbaec8b94fa: hlt                       ;...f4
  0x00007fbaec8b94fb: hlt                       ;...f4
  0x00007fbaec8b94fc: hlt                       ;...f4
  0x00007fbaec8b94fd: hlt                       ;...f4
  0x00007fbaec8b94fe: hlt                       ;...f4
  0x00007fbaec8b94ff: hlt                       ;...f4
[Exception Handler]
[Stub Code]
  0x00007fbaec8b9500: jmpq   0x00007fbaec80f860  ;...e95b63f5 ff
                                                ;   {no_reloc}
[Deopt Handler Code]
  0x00007fbaec8b9505: callq  0x00007fbaec8b950a  ;...e8000000 00
  0x00007fbaec8b950a: subq   $0x5,(%rsp)        ;...48832c24 05
  0x00007fbaec8b950f: jmpq   0x00007fbaec7eac00  ;...e9ec16f3 ff
                                                ;   {runtime_call}
  0x00007fbaec8b9514: hlt                       ;...f4
  0x00007fbaec8b9515: hlt                       ;...f4
  0x00007fbaec8b9516: hlt                       ;...f4
  0x00007fbaec8b9517: hlt                       ;...f4
OopMapSet contains 0 OopMaps

OS: Linux amd64
JavaVersion: 1.7.0_25

Compiled method (c2)    3101  428             MemoryBarrierDemo::setCancel (5 bytes)
 total in heap  [0x00007fbaec896b90,0x00007fbaec896d60] = 464
 relocation     [0x00007fbaec896cb0,0x00007fbaec896cc0] = 16
 main code      [0x00007fbaec896cc0,0x00007fbaec896d00] = 64
 stub code      [0x00007fbaec896d00,0x00007fbaec896d18] = 24
 oops           [0x00007fbaec896d18,0x00007fbaec896d20] = 8
 scopes data    [0x00007fbaec896d20,0x00007fbaec896d28] = 8
 scopes pcs     [0x00007fbaec896d28,0x00007fbaec896d58] = 48
 dependencies   [0x00007fbaec896d58,0x00007fbaec896d60] = 8
Decoding compiled method 0x00007fbaec896b90:
Code:
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} 'setCancel' '()V' in 'MemoryBarrierDemo'
  #           [sp+0x20]  (sp of caller)
  0x00007fbaec896cc0: sub    $0x18,%rsp         ;...4881ec18 000000
  0x00007fbaec896cc7: mov    %rbp,0x10(%rsp)    ;...48896c24 10
  0x00007fbaec896ccc: mov    $0xebd9deb8,%r10   ;...49bab8de d9eb00
                                                ;...000000
                                                ;   {oop(a 'java/lang/Class' = 'MemoryBarrierDemo')}
  0x00007fbaec896cd6: movb   $0x1,0x70(%r10)    ;...41c64270 01
  0x00007fbaec896cdb: lock addl $0x0,(%rsp)     ;...f0830424 00
                                                ;*putstatic cancel
                                                ; - MemoryBarrierDemo::setCancel@1 (line 5)
  0x00007fbaec896ce0: add    $0x10,%rsp         ;...4883c410
  0x00007fbaec896ce4: pop    %rbp               ;...5d
  0x00007fbaec896ce5: test   %eax,0x9c71315(%rip)        # 0x00007fbaf6508000
                                                ;...85051513 c709
                                                ;   {poll_return}
  0x00007fbaec896ceb: retq                      ;...c3
  0x00007fbaec896cec: hlt                       ;...f4
  0x00007fbaec896ced: hlt                       ;...f4
  0x00007fbaec896cee: hlt                       ;...f4
  0x00007fbaec896cef: hlt                       ;...f4
  0x00007fbaec896cf0: hlt                       ;...f4
  0x00007fbaec896cf1: hlt                       ;...f4
  0x00007fbaec896cf2: hlt                       ;...f4
  0x00007fbaec896cf3: hlt                       ;...f4
  0x00007fbaec896cf4: hlt                       ;...f4
  0x00007fbaec896cf5: hlt                       ;...f4
  0x00007fbaec896cf6: hlt                       ;...f4
  0x00007fbaec896cf7: hlt                       ;...f4
  0x00007fbaec896cf8: hlt                       ;...f4
  0x00007fbaec896cf9: hlt                       ;...f4
  0x00007fbaec896cfa: hlt                       ;...f4
  0x00007fbaec896cfb: hlt                       ;...f4
  0x00007fbaec896cfc: hlt                       ;...f4
  0x00007fbaec896cfd: hlt                       ;...f4
  0x00007fbaec896cfe: hlt                       ;...f4
  0x00007fbaec896cff: hlt                       ;...f4
[Exception Handler]
[Stub Code]
  0x00007fbaec896d00: jmpq   0x00007fbaec80f860  ;...e95b8bf7 ff
                                                ;   {no_reloc}
[Deopt Handler Code]
  0x00007fbaec896d05: callq  0x00007fbaec896d0a  ;...e8000000 00
  0x00007fbaec896d0a: subq   $0x5,(%rsp)        ;...48832c24 05
  0x00007fbaec896d0f: jmpq   0x00007fbaec7eac00  ;...e9ec3ef5 ff
                                                ;   {runtime_call}
  0x00007fbaec896d14: hlt                       ;...f4
  0x00007fbaec896d15: hlt                       ;...f4
  0x00007fbaec896d16: hlt                       ;...f4
  0x00007fbaec896d17: hlt                       ;...f4
OopMapSet contains 0 OopMaps

flag cancel set to true
Done!
yuan@yuan-HP:~/learn$
```

摘出我们所关心的部分：

``` plain
  0x00007fbaec896ccc: mov    $0xebd9deb8,%r10   ;...49bab8de d9eb00
                                                ;...000000
                                                ;   {oop(a 'java/lang/Class' = 'MemoryBarrierDemo')}
  0x00007fbaec896cd6: movb   $0x1,0x70(%r10)    ;...41c64270 01
  0x00007fbaec896cdb: lock addl $0x0,(%rsp)     ;...f0830424 00
```

在`movb   $0x1,0x70(%r10)`将`cancel`设为`true`之后有一行带有LOCK语义的代码，这就与笔者推理的一致了^_^。

（在这里还有一个让笔者困惑的地方：对于没有volatile的版本，如果执行的时候加了`-Xcomp`，程序同样可以正常退出，程序行为与加了volatile的版本一致，但是汇编代码中到LOCK。笔者对于JIT完全不懂，所以也只能暂且存疑啦。）

再回过头来整理一下思路。在笔者提供的例子主要是因为没有遵守Java Memory Model，编译器为了性能而做出的优化造成程序无法终止。这其实也不是笔者想要看到的结果，但是通过读_Intel 64 and IA-32 Architectures Software Developer's Manual Combined Volumes 3A, 3B, and 3C: System Programming Guide_解开了笔者心中的疑惑。

让笔者产生困惑的地方是buffer。

## Buffer

笔者之前头脑中的CPU模型大概是这样：

``` plain
+-------------------+
| Memory Controller |
+-------------------+
        |
+---------------------------------------------------------------+
| L3                                                            |
+---------------------------------------------------------------+
        |                        |
+-----------------+      +-----------------+
| L1 L2           |      | L1 L2           |    ...
+-----------------+      +-----------------+
        |                        |
+-----------------+      +-----------------+
| Execution Units |      | Execution Units |    ...
+-----------------+      +-----------------+
```

从执行单元进入L1或L2 cache之后就是cache coherency protocol控制的“天下”了，其它的core肯定就会“看到”相应的改变，怎么还会“看不到”相应的改变呢？这就是让笔者一直困惑的地方。（关于cache coherency，请看*支撑处理器的技术*的第5.2节）

实际上现代的CPU在执行单元与L1、L2之间还存在Load Buffer、Store Buffer和WCBuffers。它们存在的原因？可想而知是为了弥补执行单元与缓存之间的速度差异，比如[Write Combining](http://mechanical-sympathy.blogspot.com/2011/07/write-combining.html)技术。对于这三个buffer，x86指令集中有三个细粒度的指令LFENCE、SFENCE和MFENCE控制，在前面所看到的LOCK等同于MFENCE。

在*Intel 64 and IA-32 Architectures Software Developer's Manual Combined Volumes 3A, 3B, and 3C: System Programming Guide*的第8-16页：

> The SFENCE, LFENCE, and MFENCE instructions provide a performance-efficient way of ensuring load and store memory ordering between routines that produce weakly-ord ered results and routines that consume that data. The functions of these instructions are as follows:

> * SFENCE — Serializes all store (write) operations that occu rred prior to the SFENCE instruction in the program instruction stream, but does not affect load operations.

> * LFENCE — Serializes all load (read) operations that occurred prior to the LFENCE instruction in the program instruction stream, but does not affect store operations.

> * MFENCE — Serializes all store and load operations that occurred prior to the MFENCE instruction in the program instruction stream.

关于这三个指令与Java语言的关系，请看[Memory Barriers/Fences](http://mechanical-sympathy.blogspot.com/2011/07/memory-barriersfences.html)。这篇文章中还提到了Out of Order的影响，所以本文就不再重复了。

## 总结

1. CPU的“人生价值”就是尽可能的快！为此可以不择手段，比如Write Combining和Out of Order；但是它也提供了LOCK、XCHG、LFENCE、SFENCE、MFENCE等指令满足上层软件同步的需求。

2. 同步要以牺牲性能为代价。JVM会尽可能优化性能，如果要实现必要的同步就得遵守Java Memory Model，在程序中显式的使用相关机制。

3. "Shared mutability is pure evil"（引自_Programming Concurrency on the JVM_）。如果不是legacy系统，笔者肯定优先考虑Actor-based Model。
