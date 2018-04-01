

# Report of Lab 01

## 实验题目

**1、Linux的安装与使用**

**2、调试操作系统的启动过程**

## 一、Linux的安装与使用

**Linux版本：Ubuntu 16.04 LTS**

**安装在VMware Fusion上**

**在编写符合实验要求的bash文件时需要注意read的用法：(参考代码1中使用了read)**

 **bash中的while read LINE并不是一次读入整个文件, 而是open文件拿到fd之后, 在while的每一次中, read都读满缓冲区, 然后lseek记录读到的底一个\n偏移量位置, 然后write这一段内容, 进行下一次while。**

**在while read中还内嵌了read的时候, 第二次的读, 就变成了while中取用户输入的read,  然后继续while的过程，所以可能会导致执行结果与预期结果不符。**  

## 二、调试操作系统的启动过程

**调试平台(Linux版本)：Ubuntu 16.04 LTS**

**调试的内核： Linux-3.18.6**

**调试工具：qemu2.5.0+gdb7.11.1**

**调试过程：**

**第一部分：调试环境搭建**

**1、安装qemu**

**2、在Linux-Kernel官网上下载内核源代码(version:3.18.6)**

**官网下载速度太慢，可以使用mirrors.ustc.edu.cn下载。**

**3、编译内核**

**在实验中遇到过2次编译错误，均是在编译vmlinux时显示error137，第一次没有理会继续进行实验，但在用gdb调试过程中发现符号表始终不能导入，导致基本调试命令如list、break、continue、step不起作用，第二次实验再次遇到此问题后才发现vmlinux编译对后面调试的重要性。在make menuconfig时选择compile the kernel with debug info后重新编译内核问题解决。**

**内核编译时间较长，如果是用安装在虚拟机上的Linux作为平台，可以多分几个内核给Linux加快编译速度。**

**4、制作根文件系统**

**在用gcc编译inktable.c、 menu.c、 test.c这三个文件时，blog中用到了-lpthread，但在实际执行过程中会报错，将其改成-pthread之后编译成功。**

**至此调试环境搭建完成**

**第二部分：调试**

**1、用qemu启动内核**

**注意应在qemu命令后加上-S使内核暂停在刚刚开始启动的时刻便于调试。**

**2、进入gdb，输入相关命令连接内核准备开始调试**

**3、使用list命令查看内核在最开始启动时执行的代码**

**经过多次list我们可以发现，内核在最初启动的时候执行的不是c代码而是汇编代码。**

**因为对汇编语言不是很熟悉，所以大多数代码很难看懂，只能借助注释理解，通过阅读代码之间的注释可以发现，这一大段汇编代码的作用是初始化必要环境，如page tables、PDE、fixmap area等。**

**在汇编代码执行过程中，比较重要的应该就是两种模式的切换：**

**实模式：**

**内核刚刚开始执行的时候，有这一段注释：**

```c
/*
 * 64-bit kernel entrypoint; only used by the boot CPU.  On entry,
 * %esi points to the real-mode code as a 64-bit pointer.
 * CS and DS must be 4 GB flat segments, but we don't depend on
 * any particular GDT layout, because we load our own as soon as we
 * can.
 */
```

**从中我们可以知道%esi寄存器指向实模式，说明内核一开始启动的时候是出于实模式的。进入实模式后的主要执行setup代码，初始化一些必要的环境。**

**当setup基本执行完成后，出现了这一段注释：**

```c
/*
 * We want to start out with EFLAGS unambiguously cleared. Some BIOSes leave
 * bits like NT set. This would confuse the debugger if this code is traced. So
 * initialize them properly now before switching to protected mode. That means
 * DF in particular (even though we have cleared it earlier after copying the
 * command line) because GCC expects it.
 */
```

**这段注释告诉我们内核即将进入保护模式。**

**在保护模式下内核会检测位宽，如果不是16位它将重新执行setup完成初始化。**

**4、看完汇编代码，直接跳转到c代码最开始的函数start_kernel**

**在跳转过程中，我们可以看到qemu的窗口显示了start_kernel之前内核做的事情：decompress kernel和booting the kernel。**

**5、从start_kernel开始一步步调试内核**

**在start_kernel函数最开始会执行set_task_stack_end_magic(&init_task);**

**在这个函数中我们发现有一个参数init_task，经过寻找我们发现init_task的代码：**

```c
static struct signal_struct init_signals = INIT_SIGNALS(init_signals);
static struct sighand_struct init_sighand = INIT_SIGHAND(init_sighand);

/* Initial task structure */
struct task_struct init_task = INIT_TASK(init_task);
EXPORT_SYMBOL(init_task);

/*
 * Initial thread structure. Alignment of this is handled by a special
 * linker map entry.
 */
union thread_union init_thread_union __init_task_data =
	{ INIT_THREAD_INFO(init_task) };
```

**从代码中我们可以发现这是Linux在开始启动时创造的第一个进程(Process 0)，它由宏定义INIT_TASK初始化。这是一个有内核从无到有生成的一个进程。**

**继续调试~**

**可以发现，最开始知行的几个函数均是对CPU的初始化：**

```c
	boot_cpu_init();
	page_address_init();
	pr_notice("%s", linux_banner);
	setup_arch(&command_line);
	mm_init_cpumask(&init_mm);
	setup_command_line(command_line);
	setup_nr_cpu_ids();
	setup_per_cpu_areas();
	smp_prepare_boot_cpu();	/* arch-specific boot-cpu hooks */
```

**继续调试~**

**发现函数trap_init()，这个函数的作用是设置中断向量表，为后面允许中断发生做准备。**

**trap_init()后面执行的是sched_init()，内核开始初始化调度器，这个函数的代码很长，经过注释和查找资料，这个函数的主要工作是先初始化0进程，包括段选择符，描述符GDT、LDT等。然后将其他63个进程的的段选择符，描述符GDT、LDT 置空，设置好后好后，将任务0的段选择符，描述符GDT、LDT等加载进个寄存器中。接着设置系统中断定时器，中断函数判断是否要切换(意思就是当进程时间片已经消耗完时，调用定时器中断函数判断是否要切换)，最后定义系统调用。**

**由此可见这个函数在内核启动时的重要性。**

**继续调试~**

**接下来的函数都是对时钟等进行初始化。**

**当执行完函数console_init()之后，qemu的窗口出现大量信息(前面函数执行时的日志)，窗口初始化完成。此时窗口会实时显示信息。但是，源代码中在这一函数之前有这样的注释：**

```c
/*
	 * HACK ALERT! This is early. We're enabling the console before
	 * we've done PCI setups etc, and console_init() must be aware of
	 * this. But we do want output early, in case something goes wrong.
	 */
```

**提醒我们提前启动console会给系统带来风险？？**

**继续调试~**

**执行完函数sched_clock_init()后console上打印出Detected 2899.639MHZ processor，从这一刻开始，调度时间才能被获取，日志上的时间显示不再为0.000000。**

**在此之后执行的一系列函数都是为创建第一个用户进程作准备：**

```c
/* Should be run before the first non-init thread is created */
	init_espfix_bsp();
#endif
	thread_info_cache_init();
	cred_init();
	fork_init(totalram_pages);
	proc_caches_init();
	buffer_init();
	key_init();
	security_init();
	dbg_late_init();
	vfs_caches_init(totalram_pages);
	signals_init();
```

**最后执行的函数是rest_init()，这个函数建立了第一个用户进程，下面具体分析：**

```c
static noinline void __init_refok rest_init(void)
{
	int pid;

	rcu_scheduler_starting();
	/*
	 * We need to spawn init first so that it obtains pid 1, however
	 * the init task will end up wanting to create kthreads, which, if
	 * we schedule it before we create kthreadd, will OOPS.
	 */
	kernel_thread(kernel_init, NULL, CLONE_FS);
	numa_default_policy();
	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
	rcu_read_lock();
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
	rcu_read_unlock();
	complete(&kthreadd_done);

	/*
	 * The boot idle thread must execute schedule()
	 * at least once to get things moving:
	 */
	init_idle_bootup_task(current);
	schedule_preempt_disabled();
	/* Call into cpu_idle with preempt disabled */
	cpu_startup_entry(CPUHP_ONLINE);
}
```

**可以看见kernel_thread创建了init进程，查找它的代码：**

```c
pid_t kernel_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
	return do_fork(flags|CLONE_VM|CLONE_UNTRACED, (unsigned long)fn,
		(unsigned long)arg, NULL, NULL);
}
```

**可知kernel_thread调用了kernel_init，猜测kernel_thread可能通过调用kernel_init来创建进程。**

**查找kernel_init的代码，发现如下代码段：**

```c
	/*
     * We try each of these until one succeeds.
     *
     * The Bourne shell can be used instead of init if we are
     * trying to recover a really broken machine.
     */
    if (execute_command) {
        ret = run_init_process(execute_command);
        if (!ret)
            return 0;
        pr_err("Failed to execute %s (error %d).  Attempting defaults...\n",
            execute_command, ret);
    }
    if (!try_to_run_init_process("/sbin/init") ||
        !try_to_run_init_process("/etc/init") ||
        !try_to_run_init_process("/bin/init") ||
        !try_to_run_init_process("/bin/sh"))
        return 0;
```

**这个函数会去根文件系统中寻找init程序，具体路径由U-boot的环境变量bootargs提供，一旦init程序被找到，就会启动init进程，然后操作系统正式运行。**

**如果U-boot的环境变量bootargs没有传过来路径，或者路径中找不到，或者执行出错，那么kernel还留了一手以防万一。函数尾部有四个run_init_process函数，它们会去4个地方看看有没有init程序，如果以上都不成功，就启动失败了。**

**在rest_init函数最后有一个函数cpu_startup_entry(CPUHP_ONLINE)，这个函数中包含一个函数cpu_idle_loop()，会使idle进程进入死循环状态，Linux系统从内核态切换到用户态，内核的启动完成。**

**调试过程至此结束。**

**总结：**

**整个内核启动过程中比较关键的事件在汇编代码部分应该是实模式到保护模式的转换，c代码部分应该是idle进程的建立到init进程的建立(内核态到用户态的切换)，中间可能还会有一些较为关键的时间比如CPU初始化、调度初始化等。**







