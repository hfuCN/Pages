---
title: linux c 开发之编写 makefile
layout: post
guid: urn:uuid:50331d87-1a42-480d-955c-624f9d4918f1
tags:
  - linux
  - c
---

对程序员来说，编写 makefile 是一道绕不过去的坎儿。对于习惯使用各类 IDE 的程序员来说，是否会编写 makefile 显得无所谓，毕竟工具本身已经为我们做好了全部的编译流程。

但是在 Linux 上，一切都变的不一样了，没有人会为你做这一切。编码要靠自己，测试要靠自己，最有自动化编译设计也要靠自己。想想看，如果你下载了一个开源代码，却因为自动化编译失败，或许会打击你学习代码的信心，或许你会觉得这个项目不怎么样。

所以，我的理解是，我们要会编写 makefile，至少会写简单的 makefile。

首先，编写 test.h 头文件：

```
#ifndef _TEST_H
#define _TEST_H

int add(int a, int b);
int sub(int a, int b);
#endif
```

然后编写一个 add.c 文件：

```
#include "test.h"
#include <stdio.h>

int add(int a, int b)
{
        return a + b;
}

int main()
{
        printf(" 2 + 3 = %d\n", add(2, 3));
        printf(" 2 - 3 = %d\n", sub(2, 3));
        return 1;
}
```
再编写一个 sub.c：

```
#include "test.h"

int sub(int a, int b)
{
        return a - b;
}
```
那么，就这么简单的三个文件，该如何编写 makefile 呢？

```
test: add.o sub.o
        gcc -o test add.o sub.o

add.o: add.c test.h
        gcc -c add.c

sub.o: sub.c test.h
        gcc -c sub.c

clean:
        rm -rf test
        rm -rf *.o
```
此时，我们只需要执行 make 命令就可以得到编译后的二进制程序了。
