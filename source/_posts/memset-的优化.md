---
title: memset 的优化
abbrlink: 3178013999
date: 2019-11-07 06:43:26
tags:
---
工作中遇到一个问题，在CPU主频很低的情况下，memset/memcpy 等函数的执行耗时相当大。需要进行性能优化。
<!--more-->

最开始项目代码里实现的版本很简单：
```c
void *memset (void *dst, int c, size_t n) 
{
    while (n > 0)
    {
        *dst++ = c;
        n--;
    }
    return 0;//do not use this value
}
```

最开始的想法是在一个while循环中清空多个字节，减少循环次数，即：
```c
void *memset_op (void *dst, int c, size_t n) 
{ 
    char* dstp = dst;

    if (len >= 8)
    {
        size_t xlen;
 
        xlen = len / 8;
        while (xlen > 0)
        {
              dstp + 0 = c;
              dstp + 1 = c;
              dstp + 2 = c;
              dstp + 3 = c;
              dstp + 4 = c;
              dstp + 5 = c;
              dstp + 6 = c;
              dstp + 7 = c;
              
              dstp += 8 ;
              xlen -= 1;
        }
    } 
    len = len % 8;
    /* 最后8字节 */
    while (len > 0)
    {
        *dstp++ = c;
        len--;
    }
      return 0; //do not use
}
```
然后测试了下,发现提速并不是很理想。猜测可能的主要的耗时是在写入耗时，而不是循环次数。然后去查了下 glibc 的 string.h 里 memset 的实现，发现也是在一次循环中将8个内存地址赋值。实测下来性能还是达不到要求。经过一番思索（google），找到了一个可能可行的方案。目前 memset 经过 gcc 编译之后，通过调用通用寄存器来往内存写值。可以通过汇编实现 memset ,将通用寄存器改为效率更高的浮点寄存器。于是第二版优化的 memset 出炉：

```armasm
memset：
    dup     v0.16B, w1
    add     x4, x0, x2
    //if x2 > 96 , go to  sset
sset:
    and     w1, w1, 255
    bic     x3, x0, 15
    str     q0, [x0]
    sub     x2, x4, x3 
    add     x3, x3, 16
    sub     x2, x2, 64 + 16
loop:   
    stp     q0, q0, [x3], 64
    stp     q0, q0, [x3, -32]
    subs    x2, x2, 64
    b.hi    loop
    stp     q0, q0, [x4, -64]
    stp     q0, q0, [x4, -32]
    ret
  ```
利用浮点寄存器q0，快速初始化一段内存空间。实测提速有170倍。已经达到优化目标。

不过在查找资料的过程中，有博客指出 glibc 中的实现也是类似汇编。于是又去 glibc 源码里搜索了一下,最终发现 glibc 中有两处 memset 的实现。第一处即 string 目录下的 memset，采用 C 语言实现。还有在 sysdeps 目录下针对每种硬件架构，有一个 memset.S 汇编，也是采用了浮点寄存器加速的方法。当硬件支持相关特性时，该函数将代替 string 目录下的 memset.c。glibc 的 memset.S 除了采用浮点加速之外，还采用了 cache 加速。 memset.S 文件太长这里就不贴了，感兴趣的同学可以自己去看下，arm 平台和 x86 平台都有对应的汇编优化方法，同时，linux 源码的 arch 目录下也有类似的汇编优化的 memset 函数。

刚开始用汇编优化的时候我还疑惑为什么 glibc 没采用这种方式，原来是我没有找对地方。