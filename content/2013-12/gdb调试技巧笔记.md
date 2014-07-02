Title: gdb调试技巧笔记
Date: 2013-12-15
Catagory: gdb,调试
Tag: gdb,调试

GDB最基本的几条调试命令不需要再记录。记录些稍微高级点的特性。


	1. 调试带参数的程序

在使用gdb调试带参数的程序时，可以使用--args选项。


	2. GDB调试宏。C代码在预处理后，宏会被替换，导致在调试最终的可执行文件时，无法找到宏定义。

gdb编译时，打开-g开关，可以生成调试信息。其实，-g也是有级别(level)的。-g1包含非常少的
调试信息，包含函数及外部变量信息，但是没有行号及局部变量信息。-g2为默认级别，包含行号等
信息。-g3包含更多的信息，比如程序中出现的宏定义信息。macro expand命令和扩展宏，对于调试宏
非常有帮助。可是使用print显示宏替换过后的值，如果宏里面包含有编译时语句，则无法显示出值，比如
宏定义中有static_assert(C11特性,GNU扩展为_Static_assert）。

	:::c
	localhost tools # gdb --args ./kmod -V
	GNU gdb (Gentoo 7.6.2 p1) 7.6.2
	Copyright (C) 2013 Free Software Foundation, Inc.
	License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
	This is free software: you are free to change and redistribute it.
	There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
	and "show warranty" for details.
	This GDB was configured as "x86_64-pc-linux-gnu".
	For bug reporting instructions, please see:
	<http://bugs.gentoo.org/>...
	Reading symbols from /home/wolf/sourcecode/tmp/kmod-15/tools/kmod...done.
	(gdb) list
	154		}
	155	
	156		return -ENOENT;
	157	}
	158	
	159	int main(int argc, char *argv[])
	160	{
	161		int err;
	162	
	163		if (strcmp(program_invocation_short_name, "kmod") == 0)
	(gdb) b 163
	Breakpoint 1 at 0x4028a6: file tools/kmod.c, line 163.
	(gdb) next
	The program is not being run.
	(gdb) r
	Starting program: /home/wolf/sourcecode/tmp/kmod-15/tools/./kmod -V
	warning: Could not load shared library symbols for linux-vdso.so.1.
	Do you need "set solib-search-path" or "set sysroot"?

	Breakpoint 1, main (argc=2, argv=0x7fffffffe0b8) at tools/kmod.c:163
	163		if (strcmp(program_invocation_short_name, "kmod") == 0)
	(gdb) s
	164			err = handle_kmod_commands(argc, argv);
	(gdb) st
	Ambiguous command "st": stack, start, status, step, stepi, stepping, stop, strace.
	(gdb) s
	handle_kmod_commands (argc=2, argv=0x7fffffffe0b8) at tools/kmod.c:92
	92		int err = 0;
	(gdb) s
	98			c = getopt_long(argc, argv, options_s, options, NULL);
	(gdb) macro expand ARRAY_SIZE(kmod_cmds)
	expands to: (sizeof(kmod_cmds) / sizeof((kmod_cmds)[0]) + ({ _Static_assert((!__builtin_types_compatible_p(typeof(kmod_cmds), typeof(&(kmod_cmds)[0]))), "!__builtin_types_compatible_p(typeof(kmod_cmds), typeof(&(kmod_cmds)[0]))"); 0; }))
	(gdb) help macro
	Prefix for commands dealing with C preprocessor macros.

	List of macro subcommands:

	macro define -- Define a new C/C++ preprocessor macro
	macro expand -- Fully expand any C/C++ preprocessor macro invocations in EXPRESSION
	macro expand-once -- Expand C/C++ preprocessor macro invocations appearing directly in EXPRESSION
	macro list -- List all the macros defined using the `macro define' command
	macro undef -- Remove the definition of the C/C++ preprocessor macro with the given name

	Type "help macro" followed by macro subcommand name for full documentation.
	Type "apropos word" to search for commands related to "word".
	Command name abbreviations are allowed if unambiguous.

还可以使用info macros查看程序定义的所有宏。
	
	3.设置gdb命令的输出。
有时gdb命令的输出太长，在终端下滚屏都嫌太长。可以使用日志功能。

	:::c
	(gdb) set logging file /tmp/txt
	(gdb) set logging on
	Copying output to /tmp/txt.
	(gdb) set logging off
	Done logging to /tmp/txt.







链接文档:

1.<http://www.ibm.com/developerworks/cn/aix/library/au-gdb.html>
