# Debug kernel hang, oops, crash and dead lock etc.
* Menuconfig -> kernel hacking
	CONFIG_DEBUG_*
* Printk
	echo 7 > /proc/sys/kernel/printk
* proc

* strace: 函数调用的参数，消耗时间，调用顺序

* oops:

* sysrq

* gdb + qemu

* kgdb

* perf : 调试性能问题

* ftrace

* kdump + crash

## Dead lock

多个进程（≥2）因为争夺资源而相互等待的一种现象，若无外力推动，将无法继续运行下去.
* spinlock: CPU spin
	* 递归使用: 同一个进程或线程中，申请自旋锁，但没有释放之前又再次申请，一定产生死锁.
	* 进程得到自旋锁后阻塞，睡眠 : 在获得自旋锁之后调用copy_from_user()、copy_to_ser()、和kmalloc（）等有可能引起阻塞的函数.
	* 在中断中使用自旋锁是可以的，应该在进入中断的时候关闭中断，不然中断再次进入的时候，中断处理函数会自旋等待自旋锁可以再次使用.
	* 中断与中断下半部共享资源和中断与进程共享资源死锁出现的情况类似.
	* 中断下半部与进程共享资源和中断与进程共享资源死锁出现的情况类似.
**自旋锁保持期间是抢占失效的（内核不允许被抢占）**
    1. 单CPU且内核不可抢占
	* 自旋锁 : Nop
    2. 单CPU且内核可抢占
	* preempt_disable/preempt_enable
    3. 多CPU且内核可抢占
	* 
* semaphore: CPU sleep

### Linux提供检测死锁的机制
* D状态死锁检测
　所谓D状态死锁：进程长时间（系统默认配置120秒）处于TASK_UNINTERRUPTIBLE睡眠状态，这种状态下进程不响应异步信号。如：进程与外设硬件的交互（如read），通常使用这种状态来保证进程与设备的交互过程不被打断，否则设备可能处于不可控的状态。
  对于这种死锁的检测linux提供的是hungtask机制，主要内容集中在Hung_task.c文件中。具体实现原理如下:
	1) 系统创建normal级别的khungtaskd内核线程，内核线程每120秒检查一次，检查的内容：遍历所有的线程链表，发现D状态的任务，就判断自最近一次切换以来是否还有切换发生，若是有，则返回。若没有发生切换，则把任务的所有调用栈等信息打印出来。
	2) hung_task_init创建一个名为khungtaskd的内核线程，内核线程的工作由watchdog来完成.
~~~c
static int __init hung_task_init(void)
{
        atomic_notifier_chain_register(&panic_notifier_list, &panic_block);

        /* Disable hung task detector on suspend */
        pm_notifier(hungtask_pm_notify, 0);

        watchdog_task = kthread_run(watchdog, NULL, "khungtaskd");

        return 0;
}
~~~
~~~c
/**
 *      atomic_notifier_chain_register - Add notifier to an atomic notifier chain
 *      @nh: Pointer to head of the atomic notifier chain
 *      @n: New entry in notifier chain
 *
 *      Adds a notifier to an atomic notifier chain.
 *
 *      Currently always returns zero.
 */
int atomic_notifier_chain_register(struct atomic_notifier_head *nh,
                struct notifier_block *n)
{
        unsigned long flags;
        int ret;

        spin_lock_irqsave(&nh->lock, flags);
        ret = notifier_chain_register(&nh->head, n);
        spin_unlock_irqrestore(&nh->lock, flags);
        return ret;
}
~~~
~~~c
/*
 * kthread which checks for tasks stuck in D state
 */
static int watchdog(void *dummy)
{
        unsigned long hung_last_checked = jiffies;

	/* set thread nice to 0, normal thread */
        set_user_nice(current, 0);

        for ( ; ; ) {
                unsigned long timeout = sysctl_hung_task_timeout_secs;  //default 120s
                unsigned long interval = sysctl_hung_task_check_interval_secs;
                long t;

                if (interval == 0)
                        interval = timeout;
                interval = min_t(unsigned long, interval, timeout);
                t = hung_timeout_jiffies(hung_last_checked, interval);
                if (t <= 0) {
                        if (!atomic_xchg(&reset_hung_task, 0) &&
                            !hung_detector_suspended)
                                check_hung_uninterruptible_tasks(timeout);
                        hung_last_checked = jiffies;
                        continue;
                }
                schedule_timeout_interruptible(t);
        }

        return 0;
}
~~~
~~~c
static long hung_timeout_jiffies(unsigned long last_checked,
                                 unsigned long timeout)
{
        /* timeout of 0 will disable the watchdog */
        return timeout ? last_checked - jiffies + timeout * HZ :
                MAX_SCHEDULE_TIMEOUT;
}
~~~
~~~c
/*
 * Check whether a TASK_UNINTERRUPTIBLE does not get woken up for
 * a really long time (120 seconds). If that happens, print out
 * a warning.
 */
static void check_hung_uninterruptible_tasks(unsigned long timeout)
{
        int max_count = sysctl_hung_task_check_count;
        unsigned long last_break = jiffies;
        struct task_struct *g, *t;

        /*
         * If the system crashed already then all bets are off,
         * do not report extra hung tasks:
         */
        if (test_taint(TAINT_DIE) || did_panic)
                return;

        hung_task_show_lock = false;
        rcu_read_lock();
        for_each_process_thread(g, t) {
                if (!max_count--)
                        goto unlock;
                if (time_after(jiffies, last_break + HUNG_TASK_LOCK_BREAK)) {
                        if (!rcu_lock_break(g, t))
                                goto unlock;
                        last_break = jiffies;
                }
                /* use "==" to skip the TASK_KILLABLE tasks waiting on NFS */
                if (t->state == TASK_UNINTERRUPTIBLE)
                        check_hung_task(t, timeout);
        }
 unlock:
        rcu_read_unlock();
        if (hung_task_show_lock)
                debug_show_all_locks();
        if (hung_task_call_panic) {
                trigger_all_cpu_backtrace();
                panic("hung_task: blocked tasks");
        }
}
~~~
~~~c
static void check_hung_task(struct task_struct *t, unsigned long timeout)
{
        unsigned long switch_count = t->nvcsw + t->nivcsw;

        /*
         * Ensure the task is not frozen.
         * Also, skip vfork and any other user process that freezer should skip.
         */
        if (unlikely(t->flags & (PF_FROZEN | PF_FREEZER_SKIP)))
            return;

        /*
         * When a freshly created task is scheduled once, changes its state to
         * TASK_UNINTERRUPTIBLE without having ever been switched out once, it
         * musn't be checked.
         */
        if (unlikely(!switch_count))
                return;

        if (switch_count != t->last_switch_count) {
                t->last_switch_count = switch_count;
                t->last_switch_time = jiffies;
                return;
        }
        if (time_is_after_jiffies(t->last_switch_time + timeout * HZ))
                return;

        trace_sched_process_hang(t);

        if (sysctl_hung_task_panic) {
                console_verbose();
                hung_task_show_lock = true;
                hung_task_call_panic = true;
        }

        /*
         * Ok, the task did not get scheduled for more than 2 minutes,
         * complain:
         */
        if (sysctl_hung_task_warnings) {
                if (sysctl_hung_task_warnings > 0)
                        sysctl_hung_task_warnings--;
                pr_err("INFO: task %s:%d blocked for more than %ld seconds.\n",
                       t->comm, t->pid, (jiffies - t->last_switch_time) / HZ);
                pr_err("      %s %s %.*s\n",
                        print_tainted(), init_utsname()->release,
                        (int)strcspn(init_utsname()->version, " "),
                        init_utsname()->version);
                pr_err("\"echo 0 > /proc/sys/kernel/hung_task_timeout_secs\""
                        " disables this message.\n");
                sched_show_task(t);
                hung_task_show_lock = true;
        }

        touch_nmi_watchdog();
}
~~~
