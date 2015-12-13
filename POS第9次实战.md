# POS 第九次实战

## 实战题目
1. 应用程序如何感知内核中有perf类评测系统的运行（2分）；
2. 不修改UnixBench的前提下，如何让其得分分别增加一倍、减少一半（2分）。

## 实战一

### 讲评

直接从应用程序的角度出发，似乎并没有什么特别好的方法。

### 思路

模块创建文件`/proc/perf_detect`，应用程序将自己的PID写入该文件。
模块定义该文件的读写，每次写入的时候都会查看并输出。

应用程序可以通过`dmesg`查看信息来得到自己有没有被监控。

#### 问题

当然，这样存在的问题是，仍旧需要内核模块的参与，不是完全在用户态下的(访问了进程的`task_struct`)。

### 附应用程序代码: app.c

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main()
{
	freopen("/proc/perf_detect", "w", stdout);
	printf("%d\n", getpid());

	/* do something */

	return 0;
}
```

### 附模块代码: perf_detect.c

```c
/*
 * perf_detect.c  
 */

#include <linux/module.h>
#include <linux/moduleparam.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/stat.h>
#include <linux/proc_fs.h>
#include <linux/sched.h>
#include <linux/uaccess.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Shuyang Shi");

static const char proc_filename[] = "perf_detect";

struct proc_dir_entry *proc_file_entry;

#define BUFSIZE 1024

static char procfs_buff[BUFSIZE];

static char ret_buff[BUFSIZE];

static unsigned long procfs_buff_size = 0;

int my_proc_read(struct file *file, 
		char __user *buf, 
		size_t size, 
		loff_t *offset)
{
	static int flag;
	int re;
	if (flag) {
		flag = 0;
		return 0;
	}
	printk(KERN_INFO "proc read\n");
	re = 4;
	copy_to_user(buf, ret_buff, re);
	flag = 1;
	printk(KERN_INFO "copy_to_user\n");
	return re;
}

static void check(unsigned long pid)
{
	struct pid *mypid = NULL;
	struct task_struct *p_cur_task;

	mypid = find_vpid(pid);
	if (mypid == NULL) {
		printk("NULL pid\n");
		return;
	}

	p_cur_task = pid_task(mypid, PIDTYPE_PID);

	if (p_cur_task->mm == NULL) {
		printk("No process with pid %lu\n", pid);
		return;
	}
	if (p_cur_task->perf_event_ctxp != NULL) {
		if (p_cur_task->perf_event_ctxp[1] != NULL) {
			printk("pid %d is currently monitored!\n", p_cur_task->pid);
			sprintf(ret_buff, "%d\n", 1);
			return;
		}
	}
	sprintf(ret_buff, "%d\n", 0);
}

int my_proc_write(struct file *file,
		const char *buff,
		unsigned long count,
		void *data)
{
	char op[32];
	unsigned long pid;

	/* get buffer size */
	procfs_buff_size = count;
	if (procfs_buff_size > BUFSIZE) 
		procfs_buff_size = BUFSIZE;

	/* write to buffer */
	if (copy_from_user(procfs_buff, buff, procfs_buff_size))
		return -EFAULT;

	strncpy(op, procfs_buff, 10);
	op[10] = 0;
	kstrtoul(op, 10, &pid);
	check(pid);

	return procfs_buff_size;
}

static const struct file_operations proc_fops = {
	.owner = THIS_MODULE,
	.read = my_proc_read,
	.write = my_proc_write,
};

/*
 * This function is called when the module is loaded
 */
int __init init_module(void)
{
	proc_file_entry = proc_create(proc_filename, 0777, NULL, &proc_fops);
	if (!proc_file_entry)
		printk(KERN_INFO "Failed to create /proc/%s\n", proc_filename);
	else
		printk(KERN_INFO "/proc/%s created.\n", proc_filename);
	return 0;
}

/*
 * This function is called when the module is unloaded
 */
void __exit cleanup_module(void)
{
	proc_remove(proc_file_entry);
	printk(KERN_INFO "perf detector exits; /proc/%s removed.\n",\
			proc_filename);
}

```


## 实战二

### 讲评

可以从时间入手，修改一系列中的内容，`vDSO`, `clock source`, `clock generator`都可以。

### 思路

考虑到调用`gettimeofday`，因此修改其返回值。但是由于`vDSO`的存在，理论上需要做两件事：

- 修改`arch/x86/vdso/vclock_gettime.c`中的`__vdso_gettimeofday`
- Hook `syscall`中的`sys_gettimeofday` (实际上不需要，因为不会用到)

`__vdso_gettimeofday`的修改(以分数加倍为例，增加的两行添加了注释):


```c
notrace int __vdso_gettimeofday(struct timeval *tv, struct timezone *tz)
{
        if (likely(tv != NULL)) {
                if (unlikely(do_realtime((struct timespec *)tv) == VCLOCK_NONE))
                        return vdso_fallback_gtod(tv, tz);
                tv->tv_usec /= 1000;
                tv->tv_sec /= 2; /* add this line to half the reported running time */
                tv->tv_usec /= 2; /* add this line to half the reported running time */
        }
        if (unlikely(tz != NULL)) {
                tz->tz_minuteswest = gtod->tz_minuteswest;
                tz->tz_dsttime = gtod->tz_dsttime;
        }

        return 0;
}
```


