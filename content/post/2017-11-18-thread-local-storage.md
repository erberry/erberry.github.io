---
title: "Thread Local Storage"
date: 2017-11-18T21:27:02+08:00
draft: true
---

## TLS(Thread Local Storage)是什么
TLS是一种在多线程时使用的技术，它可以使你的全局变量、静态变量以及局部静态、静态成员变量成为线程独立的变量，即每个线程的TLS变量之间互不影响。例如linux下的全局变量 errno，windows下的GetLastError ，线程A在设置了一个错误信息后，线程B又设置了一个错误信息，前一个线程设置的信息就被覆盖了。解决方法就是将这个全局变量设置为TLS变量，这样在用户看来errno是一个全局变量，实际上它是每个线程独立的。

visual c++声明
```c++
__declspec(thread) int number;
```
linux下声明
```c++
__thread int number;
```
## 性能问题
关于一些对TLS过度消耗的分析文章网上有不少，根本原因在于取TLS变量的方法。我们普通的全局变量可以直接通过变量名取到变量地址，而TLS则是通过线程id去一个底层维护的数组中取到地址。大体实现方式如下：
```
void* __tls_get_addr(size_t m, size_t offset)
{
  char* tls_block = dtv[thread_id][m];
  return tls_block + offset;
}
```
*[这里是TLS的实现方式，感兴趣的朋友可以看一下](https://www.akkadia.org/drepper/tls.pdf)*

## 正常使用TLS的性能对比

1. 不使用TLS的运行情况
```
#include "stdio.h"
#include "math.h"

double tlvar;
//get_value被设置为禁止编译器内联
double get_value() __attribute__ ((noinline));
double get_value()
{
  return tlvar;
}

int main()
{
  int i;
  double f=0.0;
  tlvar = 1.0;
  for(i=0; i<1000000000; i++)
  {
     f += sqrt(get_value());
  }
  printf("f = %f\n", f);
  return 1;
}
```

```
$ time ./test_no_thread
f = 1000000000.000000

real    0m5.169s
user    0m5.137s
sys     0m0.002s
```
2. 下面使用TLS测试
```
#include "stdio.h"
#include "math.h"

__thread double tlvar;
//get_value被设置为禁止编译器内联
double get_value() __attribute__ ((noinline));
double get_value()
{
  return tlvar;
}

int main()
{
  int i;
  double f=0.0;

  tlvar = 1.0;
  for(i=0; i<1000000000; i++)
  {
    f += sqrt(get_value());
  }
  printf("f = %f\n", f);
  return 1;
}
```

```
$ time ./test
f = 1000000000.000000

real    0m5.232s
user    0m5.158s
sys     0m0.007s
```

以上测试都禁止编译器内联get_value函数。因为get_value被内联后，编译器会发现它的返回值一直是不变的，然后将get_value提到循环之前。这样会导致对tlval的地址获取只有一次，失去循环测试的意义。

## 在动态库中使用TLS变量造成的性能损耗
如果在你使用的动态库中有大量使用TLS变量的情况，那将是一种灾难。因为动态库中获取TLS变量地址使用不同的方法，会多消耗一倍的时间。
```
$ cat test_main.c
#include "stdio.h"
#include "math.h"
int test();

int main()
{
   test();
   return 1;
}
```
动态库：
```
$ cat test_lib.c
#include "stdio.h"
#include "math.h"

static __thread double tlvar;
//get_value被设置为禁止编译器内联
double get_value() __attribute__ ((noinline));
double get_value()
{
  return tlvar;
}

int test()
{
  int i;
  double f=0.0;
  tlvar = 1.0;
  for(i=0; i<1000000000; i++)
  {
    f += sqrt(get_value());
  }
  printf("f = %f\n", f);
  return 1;
}
```
动态库中循环使用TLS变量运行情况：
```
$ time ./test_main
f = 1000000000.000000

real    0m9.978s
user    0m9.906s
sys     0m0.004s
```
```
$ perf record ./test_main
```
```
$ perf report --stdio
#
# Events: 10K cpu-clock
#
# Overhead         Command        Shared Object              Symbol
# ........  ..............  ...................  ..................
#
    58.05%  inet_test_main  libinet_test_lib.so  [.] test
    21.15%  inet_test_main  ld-2.12.so           [.] __tls_get_addr
    10.69%  inet_test_main  libinet_test_lib.so  [.] get_value
     5.07%  inet_test_main  libinet_test_lib.so  [.] get_value@plt
     4.82%  inet_test_main  libinet_test_lib.so  [.] __tls_get_addr@plt
     0.23%  inet_test_main  [kernel.kallsyms]    [k] 0xffffffffa0165b75
```
如上，__tls_get_addr占据了20%多的消耗。解决办法其实很简单，就是把get_value函数设置为内联，原因上面已经讲过。