---
title: CPP 中的 delete [] 魔法🪄
description: '为什么 new [] 创建的数组需要 delete [] 来进行释放？🤔'
publishDate: 2025-05-04
tags: ['C++']
heroImage: { src: './thumbnail.png' }
language: '中文'
draft: true
---

## 前言

```cpp
class Yu {
public:
  int iNumber;
  Yu() { iNumber = 1; }
  ~Yu() {} // 包含析构函数
};

int main() {
  int *a = new int;
  delete a;

  Yu *array = new Yu[10];
  delete[] array; // new [] 创建的数组必须使用 delete [] 进行删除
}
```

### 和 `C` 对比

在 `C` 中，`malloc` 和 `free` 一般成对存在，不关心申请的堆内存是一个元素还是数组，统一都是 `void *` 操作。

```c
int* array = malloc(sizeof(int) * 10);
free(array);
```

为什么在 `CPP` 中需要使用 `delete[]` 来进行特殊处理？

## 需要标注删除 delete [] 原因

这是由于数组的元素可能需要析构，而释放一个数组必须释放数组中的所有元素，那肯定不能直接一步 `free` 掉申请的堆内存的首地址啦。而且 `new Yu` 和 `new Yu[10]` 都返回的类型是同样类型的指针 `Yu*`，编译器没有办法根据类型推导出应当使用释放一个元素的方法还是释放一个数组的方法，所以只能人工使用 `delete []` 语法标注了😂。

## 数组元素数量的获取

但是又有一个问题出现了，`delete[]` 想要析构数组中的所有元素必须先知道有多少个元素，但是 `delete[]` 既不能从类型中获知（都是一个 `Yu*` 指针乱飞），又不能从分配的内存长度除以元素长度获取（`malloc` 实际上分配的内存可能会比需要的长）。难道编译器有什么魔法🪄？结果发现 `CPP` 往申请的堆内存里塞了数组长度😂。

当用户 `new type[n]` 的时候，`CPP` 会申请如下的内存 `sizeof(long long) + sizeof(type) * n` ，然后返回的指针实际上是 `&array[0]`

| long long len 数组元素个数(n) | array[0] | array[1] | ... | array[n-1] |
| ----------------------------- | -------- | -------- | --- | ---------- |

### `free` 地址问题

所以你拿着 `&array[0]` 直接去 `free(new type[n])`，你将会享受报错 释放没有申请的内存：）

```console
g++ new-delete.cpp; ./a.out
a.out(12560,0x7ff8599d8b40) malloc: **_ error for object 0x600000398278: pointer being freed was not allocated
a.out(12560,0x7ff8599d8b40) malloc: _** set a breakpoint in malloc_error_break to debug
fish: Job 1, './a.out' terminated by signal SIGABRT (Abort)
```

此时申请的堆内存的第一个元素是 `long long` 用来保存数组的元素个数，而 `delete[]` 时会再将指针偏移回去然后再析构所有元素后再 `free` 真正分配的堆指针

但是，又要说但是了，编译器会根据数组的元素是不是需要析构来决定是否会插入这个 `long long` 来保存元素个数，比方说 `new int[n]` 这类的不需要析构的元素完全就没有插入，可以直接 `free` 的，而 `delete[]` 会根据要删除的指针的类型来进行判断是否会获取 `long long` 这个字段并对元素逐一释放。

### `new` 指针不能乱修改

还有一件事，`new` 得到的的指针不能随便改然后再 `delete`，例如说：

```cpp
Yu _array = new Yu[10];
array += 1;
std::cout << ((long long *)array - 1) << std::endl;
delete[] array;
```

此时首先会输出一个奇怪的数字，这个数字表示了 delete[] 会傻傻当作需要释放的元素个数，然后对接下来的 n 个元素（这个 n 是刚才那个奇怪的数字）执行析构函数（虽然 Yu 的析构函数什么都不干，但是 cpu 有可能会循环空转任意时间段:），然后接下来就又是经典的 free 错位置了。

## 数组的地址

对数组变量取地址是完全不同的语意，具体的类型也不相同，Talk is cheap, Show the code.

```cpp
int array[100];
int *first_element_ptr = array;
int *first_element_prt_equal = &array[0];
assert(first_element_ptr == first_element_prt_equal);
int (*array_ptr)[100] = &array;
```

## 参考

[`delete[]`如何确定动态数组的元素个数？](http://www.codelearn.club/2020/03/delete)
