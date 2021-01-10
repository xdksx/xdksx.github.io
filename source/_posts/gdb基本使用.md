---

title: gdb基础
date: 2020-07-11 17:48:05
tags: gdb
categories: linux
---

1、总体
GDB可以做四种主要的事情（以及支持这些事情的其他事情）来帮助您捕获行为中的错误：
<!--more-->
1）启动程序，并指定可能影响其行为的所有内容。
2）使程序在指定条件下停止。
3）检查程序停止时发生的情况。
4）更改程序中的内容，以便您可以尝试纠正一个错误的影响，然后继续学习另一个错误。
2、一个例子：
gdb 调试运行一个程序：
$ gdb m4  --运行
(gdb) set width 70 --设置显示最大字符数，类似more
(gdb) break m4_changequote   --设置断点，断点可以是某个函数，如main
Breakpoint 1 at 0x62f4: file builtin.c, line 879.
(gdb) run
Starting program: /work/Editorial/gdb/gnu/m4/m4  --开始运行
(gdb) n   --next 不进入函数运行，运行下一行；
(gdb) s  --step 运行下一行，遇到函数会进入

(gdb) bt --backtrace 打印调用栈
(gdb) p lquote  --打印变量lquote,这个是变量名
(gdb) p 表达式，这个表达式可以是符合语法的表达式，比如赋值语句和函数调用等，是生效的；
eg (gdb) p ++a

$1 = 0x35d40 "<QUOTE>"
(gdb) l    ---list，显示源代码
533             xfree(rquote);
(gdb) Ctrl-d --退出程序
(gdb)quit  /q --退出gdb
3、经常使用分类
3.1 如何开始：
 1）gdb programname
        2）gdb programname corefile
        3)  gdb  programname pid   / gdb -p pid  (关联上一个-g编译过的正在运行的进程)
        4)  gdb --args programname arg1 2 3....
        eg:gdb --args gcc -O2 -c foo.c
        5) gdb xx     --silent/--quit /-q 不用输出信息 
3.2 在gdb中也可以使用shell指令：
(gdb) !ifconfig

3.3 设置log文件
使用技巧：
1）回车表示重复上一个命令，除了run等；
2）step 数字，可以表示步进多少
3.4 设置打印方式：
1)要更改要打印的数组元素的限制
(gdb)set print elements 10
2)是否打印数组：
       (GDB) set print array on
       (GDB) print some_array
       (GDB) set print array off
3.5 gdb的自动补全，按tab即可，甚至可以补全函数名；
3.6 gdb 命令有option ,tab键可以召唤出来；
3.7 gdb和线程；
3.8 gdb的栈概念
4  常见命令
gdb栈查看命令
backtrace [option]… [qualifier]… [count]
bt [option]… [qualifier]… [count]
(gdb) frame 3
(gdb) frame level 3
(gdb) info frame
Stack level 1, frame at 0x7fffffffda30:
向上或向下跳帧；
up n
down n 

(gdb) frame apply all p j
#0  some_function (i=5) at fun.c:4
No symbol "j" in current context.
(gdb) frame apply all -c p j
#0  some_function (i=5) at fun.c:4
No symbol "j" in current context.
打印变量：p
显示源码 l

5 参考资料

https://scc.ustc.edu.cn/zlsc/sugon/intel/debugger/cl/index.htm#commandref/gdb_mode/cmd_set_width.htm
more:
https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/gdb.html
https://sourceware.org/gdb/current/onlinedocs/gdb/

使用例子：
