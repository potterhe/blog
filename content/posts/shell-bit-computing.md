---
title: shell算术展开、按位运算
date: 2013-07-05T00:00:00+08:00
draft: false
tags: ["shell"]
slug: shell-bit-computing
---

《shell脚本学习指南》6.1.3节描述了 shell 的算术展开，其支持的运算与C语言差不多，语法 `$((...))`

```sh
$ echo $(( 3 * 4 ))
12
```

在某些场景特别方便，可以免去写程序的烦琐，如验证某些运算。
下面是验证《深入理解计算系统》练习题2.12的场景

表达式 ~0 将生成一个全1的掩码，不管机器的字大小是多少，可移植。

```sh
$ printf "%x\n" $(( ~0 ))
ffffffffffffffff
$ printf "%#x\n" $(( ~0 ))
0xffffffffffffffff
```

上面的测试显示，shell中，0按位取反后的值是64位的。
shell 的 `printf` 命令前导字符打印：《shell脚本学习指南》表7－4：printf 的标志中描述了格式参数中"#"号的意义，"#"可以用以输出前导"0x"(16进制)、"0"(8进制)

x & 0xFF 生成一个由x的最低有效字节组成的值

```sh
$ printf "%#x\n" $(( 0x89ABCDEF & 0xFF ))
0xef
$ printf "%#.8x\n" $(( 0x89ABCDEF & 0xFF ))
0x000000ef
```

以下x = 0x87654321
A.x的最低有效字节，其他位均置为0

```sh
$ printf "%#.8x\n" $(( 0x87654321 & 0xFF ))       
0x00000021
$ printf "%#.8x\n" $(( 0x87654321 & ?0xFF ))
-bash: 0x87654321 & ?0xFF : syntax error: operand expected (error token is "?0xFF ")
```

书中给出的练习题的答案是 “x & ?0xFF”，这里的"?"号经验证，shell无法正确运行。

B.除了x的最低有效字节外，其他的位置都取补，最低有效字节保持不变。

```sh
$ printf "%#x" $(( 0x87654321 ^ ~0xff))
0xffffffff789abc21
```

上面因为~0xff会生成64位的掩码，所以结果有些不符合预期，但后32位是符合预期的。

C.x的最低有效字节设置成全1，其他字节都保持不变。

```sh
$ printf "%#x" $(( 0x87654321 | 0xff ))
0x876543ff
```
