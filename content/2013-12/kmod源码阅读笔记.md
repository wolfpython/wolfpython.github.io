Title: kmod源码阅读笔记
Date: 2013-12-14
Catagory: 数据结构，算法,Linux
Tag: 数据结构，算法，Linux

kmod[^1]是处理Linux内核模块的一系列工具集,可以插入，删除，显示，检查属性，解决模块依赖和创建别名。
取代了module-init-tools。


首先看main函数:

	:::python
	int main(int argc, char *argv[])
	{
		int err;

		if (strcmp(program_invocation_short_name, "kmod") == 0)
			err = handle_kmod_commands(argc, argv);
		else
			err = handle_kmod_compat_commands(argc, argv);

		return err;
	}
	
program_invocation_short_name是GNU扩展,表示执行程序时，程序的名称(不包含路径信息)。还有一个
program_invocation_name表示程序的全部名称（包含路径信息），等价于argv[0]。program_invocation_name
使用起来比较方便，不需要硬生生的使用argv[0]。通过man
program_invocation_name可以查看相关信息。
	
	:::text
	SYNOPSIS
		   #define _GNU_SOURCE         /* See feature_test_macros(7) */
		   #include <errno.h>

		   extern char *program_invocation_name;
		   extern char *program_invocation_short_name;
		   
program_invocation_short_name声明在errno.h里:

	:::c
	#ifdef __USE_GNU

	/* The full and simple forms of the name with which the program was
	   invoked.  These variables are set up automatically at startup based on
	   the value of ARGV[0] (this works only if you use GNU ld).  */
	extern char *program_invocation_name, *program_invocation_short_name;
	#endif /* __USE_GNU */

为啥源码里判断的是__USE_GNU宏，而man手册里的是_GNU_SOURCE？在features.h里：

	:::c
	#ifdef	_GNU_SOURCE
	# define __USE_GNU	1
	#endif
	
这个扩展,在使用GCC编译时，通过-D_GNU_SOURCE定义宏_GNU_SOURCE。GCC不会默认定义这个宏，
查看GCC默认定义的宏，可以使用命令行：

	:::bash
	% gcc -dM -E - </dev/null  #-dM可以理解为dump Macro
	
程序首先比较用户是否以kmod指令触发。如果是，则调用handle_kmod_commands函数处理，可以看出kmod支持自身的命令触发，
在命令行下，查看kmod基本的帮助信息:

	:::bash
	localhost wolf # kmod
	missing command
	kmod - Manage kernel modules: list, load, unload, etc
	Usage:
		kmod [options] command [command_options]

	Options:
		-V, --version     show version
		-h, --help        show this help

	Commands:
	  help         Show help message
	  list         list currently loaded modules
	  static-nodes outputs the static-node information installed with the currently running kernel

	kmod also handles gracefully if called from following symlinks:
	  lsmod        compat lsmod command
	  rmmod        compat rmmod command
	  insmod       compat insmod command
	  modinfo      compat modinfo command
	  modprobe     compat modprobe command
	  depmod       compat depmod command

可以看出kmod本身支持三个命令，其中一个是list,列举系统当前加载的模块。如果不是以kmod执行，则调用handle_kmod_compat_commands处理。
在编译可执行文件时，创建了几个链接到kmod,运行这几个链接时，program_invocation_short_name就是链接名，这样就调用到不同的命令。和busybox
的处理原理一样。

	:::bash
	localhost wolf # ls -l `which lsmod rmmod insmod modinfo modprobe depmod`
	lrwxrwxrwx 1 root root 10 10月 18 22:50 /sbin/depmod -> /sbin/kmod
	lrwxrwxrwx 1 root root 10 10月 18 22:50 /sbin/insmod -> /sbin/kmod
	lrwxrwxrwx 1 root root 10 10月 18 22:50 /sbin/lsmod -> /sbin/kmod
	lrwxrwxrwx 1 root root 10 10月 18 22:50 /sbin/modinfo -> /sbin/kmod
	lrwxrwxrwx 1 root root 10 10月 18 22:50 /sbin/modprobe -> /sbin/kmod
	lrwxrwxrwx 1 root root 10 10月 18 22:50 /sbin/rmmod -> /sbin/kmod

handle_kmod_commands扫描命令行参数，getopt_long时GNU扩展(man 3 getopt可以查看详细的用法)。这里面的for(;;)语句完全是多余的，每一个条件
分支要么跳出，要么是return。完全可以使用如下语句替代:

	:::c
	if (c ！= -1)
	{
		switch (c) {
		case 'h':
			kmod_help(argc, argv);
			return EXIT_SUCCESS;
		case 'V':
			puts("kmod version " VERSION);
			return EXIT_SUCCESS;
		case '?':
			return EXIT_FAILURE;
		default:
			fprintf(stderr, "Error: unexpected getopt_long() value '%c'.\n", c);
			return EXIT_FAILURE;
		}
	}
	
接着遍历kmod_cmds，和传进来的参数进行比较，如果匹配上就调用相应的处理函数。先来看看ARRAY_SIZE定义，为了方便看，使用gdb的宏扩展功能。

	:::bash
	(gdb) macro expand ARRAY_SIZE(kmod_cmds)
	expands to: (sizeof(kmod_cmds) / sizeof((kmod_cmds)[0]) + ({ _Static_assert((!__builtin_types_compatible_p(typeof(kmod_cmds), typeof(&(kmod_cmds)[0]))), "!__builtin_types_compatible_p(typeof(kmod_cmds), typeof(&(kmod_cmds)[0]))"); 0; }))
	

	/* Two gcc extensions.
	* &a[0] degrades to a pointer: a different type from an array */
	#define _array_size_chk(arr) ({ \
	assert_cc(!__builtin_types_compatible_p(typeof(arr), typeof(&(arr)[0]))); \
	0; \
	})

	define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]) + _array_size_chk(arr))

通常定义ARRAY_SIZE来表示数组的大小，<em> #define ARRAY_SIZE(x) (sizeof(x) / sizeof(*(x))) （arch/x86/boot/boot.h, line 36）</em>。
但是这里的定义后面还加了个尾_array_size_chk(arr)，意思很明确，对arr进行检查。在我的系统上(Funtoo X86_64 GCC4.8),宏最终被扩展成如上
的代码，无论如何扩展，宏最终的值都时0,所以不会对数组大小有影响。__builtin_types_compatible_p是gcc内置函数[^2],用于检查两个参数的类型是否一致，
相同，值为1,不同为0。typeof返回参数类型。_Static_assert是GCC扩展，顾名思义就是静态断言，用于编译时断言，对应于C11 static_assert。这段代码的意思
就是如果数组类型如果和指向数组第一个元素的指针类型不一致，则编译通过，否则编译通不过。啥时候编译通过不了？如果传进去一个指针，而不是数组，则编译不通过。
GCC内置函数的确很强大，一般来说数组作为参数传递，会退化为指针，而内置函数不会。看一下在不支持静态编译的系统上(编译器),如何实现类似的功能：

	:::c
	#if defined(HAVE_STATIC_ASSERT)
	#define assert_cc(expr) \
		_Static_assert((expr), #expr)
	#else
	#define assert_cc(expr) \
		   do { (void) sizeof(char [1 - 2*!(expr)]); } while(0)
	#endif
\#else分支的，使用sizeof(char [1-2*!(expr)])。如果expr值为1,对应于上面的两个类型不等（传进来的assert_cc的参数带一个!），sizeof(char [1])可以顺利
编译，如果expr为0,则sizeof(char [-1])，无法通过编译。顺便提一下，sizeof已经不完全是编译时确定，在C99标准[^3]6.5..3.4-7，明确提到了运行时
（execution time)sizeof。本例中，不涉及变长参数数组。





        








[^1]:<https://www.kernel.org/pub/linux/utils/kernel/kmod/>
[^2]:<http://gcc.gnu.org/onlinedocs/gcc-3.3.6/gcc/Other-Builtins.html>
[^3]:<http://www.open-std.org/jtc1/sc22/WG14/www/docs/n1256.pdf>
