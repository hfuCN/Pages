---
title: linux c 开发
layout: post
guid: urn:uuid:bc8939c2-f69c-454c-8ef6-a1940f63414e
tags:
  - linux
  - c
---

在大多数人看来，C 语言和 Linux 是分不开的。这其中原因很多，最重要的一点我认为 Linux 本身就是 C 语言的杰作。当然，Linux 自身对 C 语言的支持也是相当到位。作为一个真正的程序员来说，如果没有在 Linux 下使用 C 语言编写过完整的程序，那么只能说他对 C 语言本身的理解还相当肤浅，对系统本身的认识也不够到位。作为程序员来说，Linux 系统为我们提供了很多理想的环境，这其中包含了：

1. 完善的编译环境，包括 gcc，as，ld 等编译，链接工具;
2. 强大的调试环境，主要是 gdb 工具;
3. 丰富的自动编译工具，主要是 make 工具;
4. 多样化的 os 选择;
5. 浩瀚的开源代码库;

下面我们看一个简单的迭代函数：

```
#include <stdio.h>

int iterate(int value)
{
    if(1 == value)
        return 1;
    return iterate(value - 1) + value;
}

int main()
{
    printf("%d\n", iterate(10));
    return 1;
}
```
此时，我们编译这段程序：

```
~ $ gcc helloworld.c -o helloworld
```
没有报错后，我们运行该程序

```
~ $ ./helloworld
```
如果一切 OK，就能看到屏幕上打印 55 这个数字。本来 1 到 10 的和就是 55，这说明我们的程序没有错误。

当然，还有一些反汇编，此时在编译的时候我们需要加参数 －g 来添加调试信息：

```
~ $ gcc helloworld.c -g -o helloworld
~ $ objdump -S -d ./helloworld
```
objdump 中的 -S 参数是为了在显示汇编代码的时候同时显示原来 C 语言代码。

```
int iterate(int value)
{
 8048374:       55                      push   %ebp
 8048375:       89 e5                   mov    %esp,%ebp
 8048377:       83 ec 08                sub    $0x8,%esp
    if(1 == value)
 804837a:       83 7d 08 01             cmpl   $0x1,0x8(%ebp)
 804837e:       75 09                   jne    8048389 <iterate+0x15>
        return 1;
 8048380:       c7 45 fc 01 00 00 00    movl   $0x1,0xfffffffc(%ebp)
 8048387:       eb 16                   jmp    804839f <iterate+0x2b>
    return iterate(value -1) + value;
 8048389:       8b 45 08                mov    0x8(%ebp),%eax
 804838c:       83 e8 01                sub    $0x1,%eax
 804838f:       89 04 24                mov    %eax,(%esp)
 8048392:       e8 dd ff ff ff          call   8048374 <iterate>
 8048397:       8b 55 08                mov    0x8(%ebp),%edx
 804839a:       01 c2                   add    %eax,%edx
 804839c:       89 55 fc                mov    %edx,0xfffffffc(%ebp)
 804839f:       8b 45 fc                mov    0xfffffffc(%ebp),%eax
}
 80483a2:       c9                      leave
 80483a3:       c3                      ret

```
