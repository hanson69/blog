---
title: strlen和sizeof
date: 2020-05-24 14:47:27
categories: C语言
tags:
- C陷阱与缺陷
- strlen
- sizeof
---


# strlen()
思考下面这段代码会输出啥？
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void main()
{
	char ch[] = {'A'};
	char pch[] = "A"; //A \0
	printf("%d\n", strlen(ch));
	printf("%d\n", strlen(pch));
}

输出：
2
1
```

调试查看内存：
```
-exec p &ch
$2 = (char (*)[1]) 0x7ffffffedff5

-exec x/10xb 0x7ffffffedff5
0x7ffffffedff5:	0x41	0x41	0x00	0x00	0x38	0xf4	0x24	0x4e
0x7ffffffedffd:	0x2b	0xe4

-exec p &pch
$6 = (char (*)[2]) 0x7ffffffedff6

-exec x/10xb 0x7ffffffedff6
0x7ffffffedff6:	0x41	0x00	0x00	0x38	0xf4	0x24	0x4e	0x2b
0x7ffffffedffe:	0xe4	0x1d
```
我们知道，strlen是用来计算字符串的长度，遇到第一个NULL('\0')结束，不包括‘\0’。从上面内存打印可以看出，strlen(ch)=2，是因为第三个字节才是\0，在不同的机器上可能结果还不一样。



# sizeof()

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void main()
{
	char s1[] = "hello";
	char *s2 = "hello";
	char s3[10] = "hello";
	printf("strlen(s1)=%d, strlen(s2)=%d, strlen(s3)=%d\n", strlen(s1), strlen(s2), strlen(s3));
	printf("sizeof(s1)=%d, sizeof(s2)=%d, sizeof(s3)=%d\n", sizeof(s1), sizeof(s2), sizeof(s3));

}

输出：
strlen(s1)=5, strlen(s2)=5, strlen(s3)=5
sizeof(s1)=6, sizeof(s2)=8, sizeof(s3)=10
```

```
sizeof是一个关键字不是函数，发生在编译时刻，所以可以用作常量表达式。
当参数分别如下时，sizeof返回的值表示的含义如下：
数组——编译时分配的数组空间大小；
指针——存储该指针所用的空间大小（在32位系统是4，在64系统是8）；
类型——该类型所占的空间大小；
对象——对象的实际占用空间大小；
函数——函数的返回类型所占的空间大小。函数的返回类型不能是void
```

