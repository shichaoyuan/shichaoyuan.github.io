---
layout: post
title:  "Java Object in Memory"
date:   2013-09-19 20:30:01
tags: java
---

在一些高性能的Java代码中，经常会看到padding，其作用是避免false sharing。

例如Disruptor中的Sequencer：

``` plain
public class Sequence
{
    static final long INITIAL_VALUE = -1L;
    private static final Unsafe UNSAFE;
    private static final long VALUE_OFFSET;

    static
    {
        UNSAFE = Util.getUnsafe();
        final int base = UNSAFE.arrayBaseOffset(long[].class);
        final int scale = UNSAFE.arrayIndexScale(long[].class);
        VALUE_OFFSET = base + (scale * 7);
    }

    private final long[] paddedValue = new long[15];
	
......
```

其中的paddedValue从功能上来说只用到了第8个long值，而为了避免false sharing初始化15个long。

false sharing很容易理解，cache line嘛，通常是64byte或者128byte。这里让笔者困惑的是——为什么是15而不是16？

关于Java Object在内存中结构，笔者在[JVMS](http://docs.oracle.com/javase/specs/jvms/se7/jvms7.pdf)中没有找到详细的解释。而在一篇博客[Java Objects Memory Structure](http://www.codeinstructions.com/2008/12/java-objects-memory-structure.html)中找到了一些关于HotSpot JVM的相关内容。

>Rule 1: every object is aligned to an 8 bytes granularity.

>Rule 2: class attributes are ordered like this: first longs and doubles; then ints and floats; then chars and shorts; then bytes and booleans, and last the references. The attributes are aligned to their own granularity.

>Rule 3: Fields that belong to different classes of the hierarchy are NEVER mixed up together. Fields of the superclass come first, obeying rule 2, followed by the fields of the subclass.

>Rule 4: Between the last field of the superclass and the first field of the subclass there must be padding to align to a 4 bytes boundary. 

>Rule 5: When the first field of a subclass is a double or long and the superclass doesn't align to an 8 bytes boundary, JVM will break rule 2 and try to put an int, then shorts, then bytes, and then references at the beginning of the space reserved to the subclass until it fills the gap.

这些Rules也解决不了笔者的困惑，不过另外几句话提供了一些线索。


>In the Sun JVM, every object (except arrays) has a 2 words header. The first word contains the object's identity hash code plus some flags like lock state and age, and the second word contains a reference to the object's class.

>Arrays have an extra header field that contain the value of the 'length' variable. The array elements follow, and the arrays, as any regular objects, are also aligned to an 8 bytes boundary.

这篇博客的描述基于32bits HotSpot JVM，我们先写个程序验证一下。

```java
import java.lang.reflect.Field;
import java.security.AccessController;
import java.security.PrivilegedExceptionAction;

import sun.misc.Unsafe;

public class ObjectsMemoryStructure {

	private final static long[] paddedValue = new long[15];
	private static final Unsafe UNSAFE;
	
	static {
		try {
			final PrivilegedExceptionAction<Unsafe> action = new PrivilegedExceptionAction<Unsafe>() {
				@Override
				public Unsafe run() throws Exception {
					Field theUnsafe = Unsafe.class
							.getDeclaredField("theUnsafe");
					theUnsafe.setAccessible(true);
					return (Unsafe) theUnsafe.get(null);
				}
			};
			UNSAFE = AccessController.doPrivileged(action);
		} catch (Exception e) {
			throw new RuntimeException("Unable to load unsafe", e);
		}
	}

	public static void main(String[] args) {
		
		for (int i = 0; i < paddedValue.length; i++) {
			paddedValue[i] = i+1;
		}
		
		for (long i = 0; i < 160; i++) {
			System.out.print(String.format("%02x ", UNSAFE.getByte(paddedValue, i) & 0xFF));
			if (((i+1) % 8) == 0)
				System.out.println();
		}
		System.out.println();
		
		VolatileLong vlong = new VolatileLong();
		for (long i = 0; i < 88; i++) {
			System.out.print(String.format("%02x ", UNSAFE.getByte(vlong, i) & 0xFF));
			if (((i+1) % 8) == 0)
				System.out.println();
		}
		System.out.println();

	}
}

class VolatileLong {
    public volatile long value = 0L;
    public long p1=1, p2=2, p3=3, p4=4, p5=5, p6=6;
}
```

在32bits HotSpot JVM下执行输出如下：

``` plain
D:\springsource\workspace-sts-3.2.0.RELEASE\test\bin>"C:\Program Files (x86)\Jav
a\jre7\bin\java.exe" ObjectsMemoryStructure
01 00 00 00 a0 46 04 3c
0f 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00
02 00 00 00 00 00 00 00
03 00 00 00 00 00 00 00
04 00 00 00 00 00 00 00
05 00 00 00 00 00 00 00
06 00 00 00 00 00 00 00
07 00 00 00 00 00 00 00
08 00 00 00 00 00 00 00
09 00 00 00 00 00 00 00
0a 00 00 00 00 00 00 00
0b 00 00 00 00 00 00 00
0c 00 00 00 00 00 00 00
0d 00 00 00 00 00 00 00
0e 00 00 00 00 00 00 00
0f 00 00 00 00 00 00 00
01 00 00 00 a8 09 a1 3b
a8 b2 09 27 eb fa 7e ef
00 00 00 00 00 00 00 00

01 00 00 00 70 6e 05 37
00 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00
02 00 00 00 00 00 00 00
03 00 00 00 00 00 00 00
04 00 00 00 00 00 00 00
05 00 00 00 00 00 00 00
06 00 00 00 00 00 00 00
01 00 00 00 40 17 05 3c
01 00 00 00 e8 f0 09 27
01 00 00 00 38 d2 03 37

```

可见上文的描述与实现相符。`private final long[] paddedValue = new long[15];`在内存中占用了136byte，不是完美的128byte。按照paddedValue的使用方式，无论136还是128没有太大的区别，但是笔者还是有点儿小小的疑问。

在64bits HotSpot JVM下执行输出如下（UseCompressedOops默认是为true）：

``` plain
proxynode@proxynode0:~/shichaoyuan$ jdk1.7.0_25/bin/java ObjectsMemoryStructure
01 00 00 00 00 00 00 00
7a 02 e8 e7 0f 00 00 00
01 00 00 00 00 00 00 00
02 00 00 00 00 00 00 00
03 00 00 00 00 00 00 00
04 00 00 00 00 00 00 00
05 00 00 00 00 00 00 00
06 00 00 00 00 00 00 00
07 00 00 00 00 00 00 00
08 00 00 00 00 00 00 00
09 00 00 00 00 00 00 00
0a 00 00 00 00 00 00 00
0b 00 00 00 00 00 00 00
0c 00 00 00 00 00 00 00
0d 00 00 00 00 00 00 00
0e 00 00 00 00 00 00 00
0f 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00
d8 16 e8 e7 0b 5e 2f f8
eb fa 7e ef 00 00 00 00

01 00 00 00 00 00 00 00
cc 3c ed e7 00 00 00 00
00 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00
02 00 00 00 00 00 00 00
03 00 00 00 00 00 00 00
04 00 00 00 00 00 00 00
05 00 00 00 00 00 00 00
06 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00
5c 56 e9 e7 01 00 00 00
```

header的格式与长度发生了变化，非arrays的object的header也变成了16byte。

在64bits HotSpot JVM下执行输出如下（将UseCompressedOops设置为false）：

``` plain
proxynode@proxynode0:~/shichaoyuan$ jdk1.7.0_25/bin/java -XX:-UseCompressedOops ObjectsMemoryStructure
01 00 00 00 00 00 00 00
d0 13 40 a4 41 7f 00 00
0f 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00
02 00 00 00 00 00 00 00
03 00 00 00 00 00 00 00
04 00 00 00 00 00 00 00
05 00 00 00 00 00 00 00
06 00 00 00 00 00 00 00
07 00 00 00 00 00 00 00
08 00 00 00 00 00 00 00
09 00 00 00 00 00 00 00
0a 00 00 00 00 00 00 00
0b 00 00 00 00 00 00 00
0c 00 00 00 00 00 00 00
0d 00 00 00 00 00 00 00
0e 00 00 00 00 00 00 00
0f 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00
c8 ba 40 a4 41 7f 00 00

01 00 00 00 00 00 00 00
38 9c 6a a4 41 7f 00 00
00 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00
02 00 00 00 00 00 00 00
03 00 00 00 00 00 00 00
04 00 00 00 00 00 00 00
05 00 00 00 00 00 00 00
06 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00
c0 e9 4a a4 41 7f 00 00
```

header的格式与长度又发生了变化，arrays的header变成了24byte。

看来这个没有被规范的地方，不同的硬件和平台存在着些许的不同。15或许在某些JVM实现中是完美的，所以笔者也不打算深究这个疑问了。

下面我们来看看header里存储了那些内容。

笔者查看了[openjdk7u](http://hg.openjdk.java.net/jdk7u/jdk7u)项目中hotspot的源代码，在`oop.hpp`和`markOop.hpp`中找到了一些相关的代码和注释。

```cpp
class oopDesc {
  friend class VMStructs;
 private:
  volatile markOop  _mark;
  union _metadata {
    wideKlassOop    _klass;
    narrowOop       _compressed_klass;
  } _metadata;

```

```cpp
// The markOop describes the header of an object.
//
// Note that the mark is not a real oop but just a word.
// It is placed in the oop hierarchy for historical reasons.
//
// Bit-format of an object header (most significant first, big endian layout below):
//
//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
//
//  - hash contains the identity hash value: largest value is
//    31 bits, see os::random().  Also, 64-bit vm's require
//    a hash value no bigger than 32 bits because they will not
//    properly generate a mask larger than that: see library_call.cpp
//    and c1_CodePatterns_sparc.cpp.
//
//  - the biased lock pattern is used to bias a lock toward a given
//    thread. When this pattern is set in the low three bits, the lock
//    is either biased toward a given thread or "anonymously" biased,
//    indicating that it is possible for it to be biased. When the
//    lock is biased toward a given thread, locking and unlocking can
//    be performed by that thread without using atomic operations.
//    When a lock's bias is revoked, it reverts back to the normal
//    locking scheme described below.
//
//    Note that we are overloading the meaning of the "unlocked" state
//    of the header. Because we steal a bit from the age we can
//    guarantee that the bias pattern will never be seen for a truly
//    unlocked object.
//
//    Note also that the biased state contains the age bits normally
//    contained in the object header. Large increases in scavenge
//    times were seen when these bits were absent and an arbitrary age
//    assigned to all biased objects, because they tended to consume a
//    significant fraction of the eden semispaces and were not
//    promoted promptly, causing an increase in the amount of copying
//    performed. The runtime system aligns all JavaThread* pointers to
//    a very large value (currently 128 bytes (32bVM) or 256 bytes (64bVM))
//    to make room for the age bits & the epoch bits (used in support of
//    biased locking), and for the CMS "freeness" bit in the 64bVM (+COOPs).
//
//    [JavaThread* | epoch | age | 1 | 01]       lock is biased toward given thread
//    [0           | epoch | age | 1 | 01]       lock is anonymously biased
//
//  - the two lock bits are used to describe three states: locked/unlocked and monitor.
//
//    [ptr             | 00]  locked             ptr points to real header on stack
//    [header      | 0 | 01]  unlocked           regular object header
//    [ptr             | 10]  monitor            inflated lock (header is wapped out)
//    [ptr             | 11]  marked             used by markSweep to mark an object
//                                               not valid at any other time
//
//    We assume that stack/thread pointers have the lowest two bits cleared.

```

在32bits的情况下，前32bits包含hash值以及锁信息，后32bit包含指向类的指针。

在64bits的情况下，前32bits包含hash值以及锁信息，后64bit包含指向类的指针，如果没有UseCompressedOops，那么长度为64bit；如果使用了UseCompressedOops，那么长度为32bit。这也就解释了第二个测试中，array的长度仍为16byte的原因，因为表示长度的32bit藏在了压缩后的指针后面。

坦白说，显式使用padding的方式很不优雅，如果能实现在JIT中对Java程序员来说绝对是福音啊。
