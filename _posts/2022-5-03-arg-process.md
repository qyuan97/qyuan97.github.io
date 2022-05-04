---
title: 命令行参数处理
date: 2022-05-04 08:00
categories: [C++, Others]
tags: [getopt getopt_long]
---

# getopt()函数和getopt_long()函数

## getopt()
getopt()用于单字母的选项，如`-a`,`-t`等，函数的声明如下:
```c++
#include <unistd.h>
int getopt(int argc, char * const argv[], const char *optstring);
extern char *optarg;
extern int optind, opterr, optopt;
```
### 参数
`argc`和`argv[]`，一般直接使用`main()`传递进来的数值。
1. `argc`: argument count, 记录了命令行参数的个数(包括命令本身)
2. `argv[]`: argument vector, 记录了命令行参数的具体内容
```
$ ./test 1 2 3
argc = 4
argv[0] = ./test
argv[1] = 1
argv[2] = 2
argv[3] = 3
```
3. optstring: 规定合法选项以及选项是否带参数，一般为合法选项字母构成的字符串，如果字母后面带上冒号`:`就说明该选项必须有参数。如`ht:`说明有两个选项`-h`和`-t`且后者`-t`必须带有参数(`-t 60`)
### 返回值
1. `option character`: 一般情况下，`getopt()`读取到合法选项，就返回该选项
2. `-1` 结束选项
`while ((opt = getopt(argc, argv, "ab:")) != -1) {……}`
3. `?`，一般情况下，遇到非法选项或者参数缺失都会返回`?`。如果需要区分这两种错误，可以在`optstring`的开头加上`:`, 如`:ht:`，这样参数缺失就返回`:`, 非法选项就返回`?`
### 相关变量
`optind`(option index):数组下标, 指向下一个未处理的参数，从`1`开始
`optarg`：如果合法选项带有参数，那么对应的参数，赋值给`optarg`

## getopt_long()
getopt_long()用于处理长选项, 如`-help`
```c++
#include <getopt.h>
int getopt_long(int argc, char *const argv[], const char *optstring, const struct option *longopts, int *longindex);
```
### 参数
`longopts`: 用于规定合法长选项以及长选项是否带有参数
```c++
struct option{
	const char *name;
	int has_arg;
	int *flag;
	int val;
}
```
1. `name`: 长选项的名称
2. `has_arg`: 参数情况
   * no_argument 0 无参数
   * required_argument 1 有参数
   * optional_argument 2 参数可选
3. 如果`flag`为`NULL`, `val`作为`getopt_long()`的返回值；如果`flag`不为`NULL`, `val`赋值给`flag`指针所指内容，同时`getopt_long()`返回`0`
4. 
```c++
static const struct option long_options[] = {
	{"version", no_arguments, NULL, 'V'},
	{"proxy", required_arguments, NULL, 'p'},
	{NULL, 0, NULL, 0} //最后一个元素必为全0
};
```
5. `longindex`: 一般设置为`NULL`; 如果不为`NULL`, 指向每次找到的长选项在`longopts`的位置，可以通过该索引找到当前长选项的具体信息。

> Reference:
[https://www.jianshu.com/p/80cdbf718916](https://www.jianshu.com/p/80cdbf718916)