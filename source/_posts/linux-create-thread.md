---
title: Linux如何创建进程
date: 2021-03-30 11:46:54
tags: 
- Linux
- thread
---

> Linux版本: 5.0
> https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.0-rc3.tar.gz

经常会遇到的面试题是: 进程和线程的区别?
<!-- more -->
进程成本高, 经常是持有资源的基本单位, etc.
那么进程到底是什么样子呢?


```
include/linux/sched.h
```

进程描述符在源码中近600行
```
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
       /*
        * For reasons of header soup (see current_thread_info()), this
        * must be the first element of task_struct.
        */
       struct thread_info            thread_info;
#endif
       /* -1 unrunnable, 0 runnable, >0 stopped: */
       volatile long                state;

       /*
        * This begins the randomizable portion of task_struct. Only
        * scheduling-critical items should be added above here.
        */
       randomized_struct_fields_start

       void                        *stack;


       ......省略n行......


       struct mm_struct              *mm;
       struct mm_struct              *active_mm;


       ......省略n行......


       /* PID/PID hash table linkage. */
       struct pid                   *thread_pid;


       ......省略n行......

       /* Filesystem information: */
       struct fs_struct              *fs;

       /* Open file information: */
       struct files_struct           *files;


       /* Signal handlers: */
       struct signal_struct          *signal;
       struct sighand_struct         *sighand;

       
       ......省略n行......

}
```

进程描述符保存了进程的状态和一些必要的信息
例如:

1. thread_info 进程的基本信息
2. mm_struct 指向内存区域描述符的指针
3. fs_struct 当前目录
4. files_struct 打开的文件
5. signal 信号
6. pid 进程的标号
7. etc


600行的结构体, 去掉注释, 也挺多字段的, 成本看上去就高.

那我们继续来看看创建进程的步骤:

```
fork.c
```

```
/* For compatibility with architectures that call do_fork directly rather than
 * using the syscall entry points below. */
long do_fork(unsigned long clone_flags,
             unsigned long stack_start,
             unsigned long stack_size,
             int __user *parent_tidptr,
             int __user *child_tidptr)
{
       return _do_fork(clone_flags, stack_start, stack_size,
                     parent_tidptr, child_tidptr, 0);
}
#endif

/*
 * Create a kernel thread.
 */
pid_t kernel_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
       return _do_fork(flags|CLONE_VM|CLONE_UNTRACED, (unsigned long)fn,
              (unsigned long)arg, NULL, NULL, 0);
}
```

如上所示, 不管是用户调用fork, vfork, pthread_create, 最后都会走到do_fork函数

创建内核线程, 最终调用的_do_fork函数


```
do_fork ---------------------|
                             |
                             --------> _do_fork ----------> copy_process
                             |
kernel_thread----------------|
```

根据函数调用最终走到copy_process函数


```
static __latent_entropy struct task_struct *copy_process(
					struct pid *pid,
					int trace,
					int node,
					struct kernel_clone_args *args)
{

    ......省略n行......
    
	p = dup_task_struct(current, node);

    
	retval = copy_files(clone_flags, p);
	retval = copy_fs(clone_flags, p);
	retval = copy_signal(clone_flags, p);
	retval = copy_mm(clone_flags, p);
	retval = copy_namespaces(clone_flags, p);
	retval = copy_io(clone_flags, p);
	retval = copy_thread(clone_flags, args->stack, args->stack_size, p, args->tls);

    	if (pid != &init_struct_pid) {
		pid = alloc_pid(p->nsproxy->pid_ns_for_children, args->set_tid,
				args->set_tid_size);
		if (IS_ERR(pid)) {
			retval = PTR_ERR(pid);
			goto bad_fork_cleanup_thread;
		}
	}
    
    ......省略n行......
}
```

简略的来看, 就是一直在copy东西.

而且也可以看到docker底层的cgroup和namespace不是什么新的东西.