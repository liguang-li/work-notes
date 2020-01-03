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
   进程等待 I/O 资源无法得到满足，长时间（系统默认配置 120 秒）处于 TASK_UNINTERRUPTIBLE 睡眠状态，这种状态下进程不响应异步信号（包括 kill -9）。如：进程与外设硬件的交互（如 read），通常使用这种状态来保证进程与设备的交互过程不被打断，否则设备可能处于不可控的状态。

  对于这种死锁的检测 Linux 提供的是 hung task 机制。触发该问题成因比较复杂多样，可能因为 synchronized_irq、mutex lock、内存不足等。D 状态死锁只是局部多进程间互锁，一般来说只是 hang 机、冻屏，机器某些功能没法使用，但不会导致没喂狗而系统死机。

	主要内容集中在Hung_task.c文件中。具体实现原理如下:
1) 系统创建normal级别的khungtaskd内核线程，内核线程每120秒检查一次，检查的内容：遍历所有的线程链表，发现D状态的任务，就判断自最近一次切换以来是否还有切换发生，若是有，则返回。若没有发生切换，则把任务的所有调用栈等信息打印出来。
	2) hung_task_init创建一个名为khungtaskd的内核线程，内核线程的工作由watchdog来完成.

	* cat /proc/sys/kernel/
		      hung_task_check_count		//120s
	        hung_task_panic			//0
	        hung_task_timeout_secs		//
		      hung_task_warnings		//6
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
		/* check whether the timeout has elapsed, hung_task_checked - jiffes + timeout * HZ */
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
		/* goto unlock if we have searched max threads,becase preempt disabled */
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
* R状态死锁检测
    进程长时间（系统默认配置60秒）处于TASK_RUNNING状态垄断CPU而不发生切换，一般情况下是进程关抢占或关中断后长时候执行任务、死循环，此时往往会导致多CPU间互锁，整个系统无法正常调度，导致喂狗线程无法执行，无法喂狗而最终看门狗复位的重启。该问题多为原子操作，spinlock等CPU间并发操作处理不当造成。

	Lockdep死锁检测工具检测的死锁类型就是R状态死锁。
	**soft lockup 和 hard lockup?**

	* soft lockup：抢占被长时间关闭而导致进程无法调度.
	* hard lockup : 中断被长时间关闭而导致更严重的问题.

**相关内核配置选项**
* CONFIG_PROVE_LOCKING
	This feature enables the kernel to report locking related deadlocks before they actually occur. For more details, see Documentation/locking/lockdep-design.txt.
* CONFIG_DEBUG_LOCK_ALLOC
	Detect incorrect freeing of live locks.
* CONFIG_DEBUG_LOCKDEP
	The lock dependency engine will do additional runtime checks to debug itself, at the price of more runtime overhead.
* CONFIG_LOCK_STAT
	Lock usage statistics. For more details, see Documentation/locking/lockstat.txt
* CONFIG_DEBUG_LOCKING_API_SELFTESTS
	The kernel to run a short self-test during bootup in start_kernel(). The self-test checks whether common types of locking bugs are detected by debugging mechanisms or not. For more details, see lib/locking-selftest.c

* CONFIG_LOCKUP_DETECTOR

* CONFIG_HARDLOCKUP_DETECTOR_OTHER_CPU

* CONFIG_HARDLOCKUP_DETECTOR

~~~c
void __init lockup_detector_init(void)
{
        if (tick_nohz_full_enabled())
                pr_info("Disabling watchdog on nohz_full cores by default\n");

        cpumask_copy(&watchdog_cpumask,
                     housekeeping_cpumask(HK_FLAG_TIMER));

        if (!watchdog_nmi_probe())
                nmi_watchdog_available = true;
        lockup_detector_setup();
}
~~~
~~~c
/*
 * Create the watchdog thread infrastructure and configure the detector(s).
 *
 * The threads are not unparked as watchdog_allowed_mask is empty.  When
 * the threads are successfully initialized, take the proper locks and
 * unpark the threads in the watchdog_cpumask if the watchdog is enabled.
 */
static __init void lockup_detector_setup(void)
{
        /*
         * If sysctl is off and watchdog got disabled on the command line,
         * nothing to do here.
         */
        lockup_detector_update_enable();

        if (!IS_ENABLED(CONFIG_SYSCTL) &&
            !(watchdog_enabled && watchdog_thresh))
                return;

        mutex_lock(&watchdog_mutex);
        lockup_detector_reconfigure();
        softlockup_initialized = true;
        mutex_unlock(&watchdog_mutex);
}
~~~
~~~c
static void lockup_detector_reconfigure(void)
{
        cpus_read_lock();
        watchdog_nmi_stop();

        softlockup_stop_all();
        set_sample_period();
        lockup_detector_update_enable();
        if (watchdog_enabled && watchdog_thresh)
                softlockup_start_all();

        watchdog_nmi_start();
        cpus_read_unlock();
        /*
         * Must be called outside the cpus locked section to prevent
         * recursive locking in the perf code.
         */
        __lockup_detector_cleanup();
}
~~~
~~~c
/* hard lockup */
void __weak watchdog_nmi_disable(unsigned int cpu)
{
        hardlockup_detector_perf_disable();
}

/**
 * hrtimer_cancel - cancel a timer and wait for the handler to finish.
 * @timer:      the timer to be cancelled
 *
 * Returns:
 *  0 when the timer was not active
 *  1 when the timer was active
 */
int hrtimer_cancel(struct hrtimer *timer)
{
        int ret;

        do {
                ret = hrtimer_try_to_cancel(timer);

                if (ret < 0)
                        hrtimer_cancel_wait_running(timer);
        } while (ret < 0);
        return ret;
}

static void watchdog_disable(unsigned int cpu)
{
        struct hrtimer *hrtimer = this_cpu_ptr(&watchdog_hrtimer);

        WARN_ON_ONCE(cpu != smp_processor_id());

        /*
         * Disable the perf event first. That prevents that a large delay
         * between disabling the timer and disabling the perf event causes
         * the perf NMI to detect a false positive.
         */
        watchdog_nmi_disable(cpu);
        hrtimer_cancel(hrtimer);
        wait_for_completion(this_cpu_ptr(&softlockup_completion));
}

static int softlockup_stop_fn(void *data)
{
        watchdog_disable(smp_processor_id());
        return 0;
}

static void softlockup_stop_all(void)
{
        int cpu;

        if (!softlockup_initialized)
                return;

        for_each_cpu(cpu, &watchdog_allowed_mask)
                smp_call_on_cpu(cpu, softlockup_stop_fn, NULL, false);

        cpumask_clear(&watchdog_allowed_mask);
}
~~~
~~~c
static void softlockup_start_all(void)
{
        int cpu;

        cpumask_copy(&watchdog_allowed_mask, &watchdog_cpumask);
        for_each_cpu(cpu, &watchdog_allowed_mask)
                smp_call_on_cpu(cpu, softlockup_start_fn, NULL, false);
}
static int softlockup_start_fn(void *data)
{
        watchdog_enable(smp_processor_id());
        return 0;
}
static void watchdog_enable(unsigned int cpu)
{
        struct hrtimer *hrtimer = this_cpu_ptr(&watchdog_hrtimer);
        struct completion *done = this_cpu_ptr(&softlockup_completion);

        WARN_ON_ONCE(cpu != smp_processor_id());

        init_completion(done);
        complete(done);

        /*
         * Start the timer first to prevent the NMI watchdog triggering
         * before the timer has a chance to fire.
         */
        hrtimer_init(hrtimer, CLOCK_MONOTONIC, HRTIMER_MODE_REL_HARD);
        hrtimer->function = watchdog_timer_fn;
        hrtimer_start(hrtimer, ns_to_ktime(sample_period),
                      HRTIMER_MODE_REL_PINNED_HARD);

        /* Initialize timestamp */
        __touch_watchdog();
        /* Enable the perf event */
        if (watchdog_enabled & NMI_WATCHDOG_ENABLED)
                watchdog_nmi_enable(cpu);
}
/* Commands for resetting the watchdog */
static void __touch_watchdog(void)
{
        __this_cpu_write(watchdog_touch_ts, get_timestamp());
}
int __weak watchdog_nmi_enable(unsigned int cpu)
{
        hardlockup_detector_perf_enable();
        return 0;
}
~~~
~~~c
/* watchdog kicker functions */
static enum hrtimer_restart watchdog_timer_fn(struct hrtimer *hrtimer)
{
        unsigned long touch_ts = __this_cpu_read(watchdog_touch_ts);
        struct pt_regs *regs = get_irq_regs();
        int duration;
        int softlockup_all_cpu_backtrace = sysctl_softlockup_all_cpu_backtrace;

        if (!watchdog_enabled)
                return HRTIMER_NORESTART;

        /* kick the hardlockup detector */
        watchdog_interrupt_count();

        /* kick the softlockup detector */
        if (completion_done(this_cpu_ptr(&softlockup_completion))) {
                reinit_completion(this_cpu_ptr(&softlockup_completion));
                stop_one_cpu_nowait(smp_processor_id(),
                                softlockup_fn, NULL,
                                this_cpu_ptr(&softlockup_stop_work));
        }

        /* .. and repeat */
        hrtimer_forward_now(hrtimer, ns_to_ktime(sample_period));

        if (touch_ts == 0) {
                if (unlikely(__this_cpu_read(softlockup_touch_sync))) {
                        /*
                         * If the time stamp was touched atomically
                         * make sure the scheduler tick is up to date.
                         */
                        __this_cpu_write(softlockup_touch_sync, false);
                        sched_clock_tick();
                }

                /* Clear the guest paused flag on watchdog reset */
                kvm_check_and_clear_guest_paused();
                __touch_watchdog();
                return HRTIMER_RESTART;
        }
        /* check for a softlockup
         * This is done by making sure a high priority task is
         * being scheduled.  The task touches the watchdog to
         * indicate it is getting cpu time.  If it hasn't then
         * this is a good indication some task is hogging the cpu
         */
        duration = is_softlockup(touch_ts);
        if (unlikely(duration)) {
                /*
                 * If a virtual machine is stopped by the host it can look to
                 * the watchdog like a soft lockup, check to see if the host
                 * stopped the vm before we issue the warning
                 */
                if (kvm_check_and_clear_guest_paused())
                        return HRTIMER_RESTART;

                /* only warn once */
                if (__this_cpu_read(soft_watchdog_warn) == true) {
                        /*
                         * When multiple processes are causing softlockups the
                         * softlockup detector only warns on the first one
                         * because the code relies on a full quiet cycle to
                         * re-arm.  The second process prevents the quiet cycle
                         * and never gets reported.  Use task pointers to detect
                         * this.
                         */
                        if (__this_cpu_read(softlockup_task_ptr_saved) !=
                            current) {
                                __this_cpu_write(soft_watchdog_warn, false);
                                __touch_watchdog();
                        }
                        return HRTIMER_RESTART;
                }

                if (softlockup_all_cpu_backtrace) {
                        /* Prevent multiple soft-lockup reports if one cpu is already
                         * engaged in dumping cpu back traces
                         */
                        if (test_and_set_bit(0, &soft_lockup_nmi_warn)) {
                                /* Someone else will report us. Let's give up */
                                __this_cpu_write(soft_watchdog_warn, true);
                                return HRTIMER_RESTART;
                        }
                }

                pr_emerg("BUG: soft lockup - CPU#%d stuck for %us! [%s:%d]\n",
                        smp_processor_id(), duration,
                        current->comm, task_pid_nr(current));
                __this_cpu_write(softlockup_task_ptr_saved, current);
                print_modules();
                print_irqtrace_events(current);
                if (regs)
                        show_regs(regs);
                else
                        dump_stack();

                if (softlockup_all_cpu_backtrace) {
                        /* Avoid generating two back traces for current
                         * given that one is already made above
                         */
                        trigger_allbutself_cpu_backtrace();

                        clear_bit(0, &soft_lockup_nmi_warn);
                        /* Barrier to sync with other cpus */
                        smp_mb__after_atomic();
                }

                add_taint(TAINT_SOFTLOCKUP, LOCKDEP_STILL_OK);
                if (softlockup_panic)
                        panic("softlockup: hung tasks");
                __this_cpu_write(soft_watchdog_warn, true);
        } else
                __this_cpu_write(soft_watchdog_warn, false);

        return HRTIMER_RESTART;
}
~~~
~~~c
/*
 * The watchdog thread function - touches the timestamp.
 *
 * It only runs once every sample_period seconds (4 seconds by
 * default) to reset the softlockup timestamp. If this gets delayed
 * for more than 2*watchdog_thresh seconds then the debug-printout
 * triggers in watchdog_timer_fn().
 */
static int softlockup_fn(void *data)
{
        __this_cpu_write(soft_lockup_hrtimer_cnt,
                         __this_cpu_read(hrtimer_interrupts));
        __touch_watchdog();
        complete(this_cpu_ptr(&softlockup_completion));

        return 0;
}
~~~


## oops
* case 1:

[ 1530.089796] BUG: unable to handle kernel NULL pointer dereference at           (null)
[ 1530.115557] IP: [<ffffffff810114e0>] hw_breakpoint_pmu_read+0x10/0x10
[ 1530.136737] PGD 1fd24fa067 PUD 1fe1cf5067 PMD 0
[ 1530.151929] Oops: 0002 [#1] SMP
[ 1530.162538] Modules linked in: tscsync nf_conntrack_ipv6 nf_defrag_ipv6 ip6table_mangle igb_uio(O) ipt_MASQUERADE iptable_nat nf_nat_ipv4 nf_nat nf_conntrack_ipv4 nf_defrag_ipv4 xt_conntrack nf_conntrack ipt_REJECT xt_CHECKSUM iptable_mangle xt_tcpudp bridge stp llc ip6table_filter ip6_tables iptable_filter ip_tables ebtable_nat ebtables x_tables iTCO_wdt iTCO_vendor_support coretemp intel_rapl iosf_mbi kvm_intel kvm aesni_intel aes_x86_64 glue_helper lrw ablk_helper cryptd lpc_ich evdev acpi_power_meter hwmon ipv6
[ 1530.313088] CPU: 52 PID: 0 Comm: swapper/52 Tainted: G           O 3.10.62-ltsi-WR6.0.0.29_standard #1
[ 1530.343685] Hardware name: Cisco Systems Inc FPR4K-SM-36/FPR4K-SM-36, BIOS FXOSSM1.1.2.1.6.072020171212 07/20/2017
[ 1530.377710] task: ffff881fe2807180 ti: ffff881fe280a000 task.ti: ffff881fe280a000
[ 1530.402303] RIP: 0010:[<ffffffff810114e0>]  [<ffffffff810114e0>] hw_breakpoint_pmu_read+0x10/0x10
[ 1530.431482] RSP: 0018:ffff88207fc43eb8  EFLAGS: 00010007
[ 1530.448930] RAX: 0000000000000000 RBX: ffffffff81a15600 RCX: 0000000000000003
[ 1530.472381] RDX: 0000000000000000 RSI: 000001646140e400 RDI: ffffffff81a15600
[ 1530.495831] RBP: ffff88207fc43ec0 R08: ffffffff8180c3e0 R09: 0000000000000008
[ 1530.519280] R10: 0000000000000009 R11: 0000000000000000 R12: 00000000000005fa
[ 1530.542729] R13: 000000000004aca8 R14: 0000000000000004 R15: 000001646140e400
[ 1530.566179] FS:  0000000000000000(0000) GS:ffff88207fc40000(0000) knlGS:0000000000000000
[ 1530.592774] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[ 1530.611651] CR2: 0000000000000000 CR3: 0000001fc7ac7000 CR4: 00000000001407e0
[ 1530.635102] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[ 1530.658553] DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
[ 1530.682002] Stack:
[ 1530.688586]  ffffffff81011529 ffff88207fc43ee8 ffffffff8108247d ffff88207fc4cb40
[ 1530.712934]  000001646140e400 0000000000000000 ffff88207fc43f10 ffffffff81088cbf
[ 1530.737284]  ffff88207fc4d260 152c7038901cf0aa ffff88207fc4d160 ffff88207fc43f20
[ 1530.761633]  ffffffff8108a48f ffff88207fc43f88 ffffffff81063d51 0000016460c7ba3c
[ 1530.785982]  ffff88207fc4d298 ffff88207fc4d258 ffff88207fc4d218 0000000300000092
[ 1530.810328]  0000016460c7ba3c ffff88207fc4cb40 0000000000000000 ffff881fe280bfd8
[ 1530.834677]  ffff881fe280bfd8 ffff881fe280bfd8 ffff88207fc43fa8 ffffffff8102cfd4
[ 1530.859025]  ffff881fe280bfd8 ffffffff81a88090 ffff881fe280bed8 ffffffff81630d9d
[ 1530.883375]  ffff881fe280be48 <EOI>  ffff881fe280bed8 ffffffff81088cf1 0000000000000000
[ 1530.909736]  0000000000000000 0000000000000418 0000000000000000 00000000ffffffed
[ 1530.934084]  0100000000000000 0140000000000000 0000000000000000 0000000000000000
[ 1530.958434]  ffffffffffffff10 ffffffff81033e26 0000000000000010 0000000000000286
[ 1530.982785]  ffff881fe280bed8 0000000000000018 ffffffff81a88090 ffff881fe280bee8
[ 1531.007132]  ffffffff810126b9 ffff881fe280bef8 ffffffff81012e96 ffff881fe280bf30
[ 1531.031479]  ffffffff81081579 0000000000000034 0000000000000000 0000000000000000
[ 1531.055827]  0000000000000000 0000000000000000 ffff881fe280bf48 ffffffff8161c0e9
[ 1531.080177]  0000000000000000 0000000000000000 0000000000000000 0000000000000000
[ 1531.104524]  0000000000000000 0000000000000000 0000000000000000 0000000000000000
[ 1531.128873]  0000000000000000 0000000000000000 0000000000000000 0000000000000000
[ 1531.153221]  0000000000000000 0000000000000000 0000000000000000 0000000000000000
[ 1531.177567]  0000000000000000 0000000000000000 ffffffffffffffff 0000000000000000
[ 1531.201913]  0000000000000010 0000000000000202 ffff881fe280bf58 0000000000000018
[ 1531.226261] Call Trace:
[ 1531.234275]  <IRQ>
[ 1531.240575]  [<ffffffff81011529>] ? read_tsc+0x9/0x20 [ 1531.257762]  [<ffffffff8108247d>] ktime_get+0x4d/0xd0
[ 1531.274354]  [<ffffffff81088cbf>] clockevents_program_event+0x3f/0x100
[ 1531.295805]  [<ffffffff8108a48f>] tick_program_event+0x1f/0x30
[ 1531.314971]  [<ffffffff81063d51>] hrtimer_interrupt+0x121/0x220
[ 1531.334420]  [<ffffffff8102cfd4>] smp_apic_timer_interrupt+0x64/0xa0
[ 1531.355303]  [<ffffffff81630d9d>] apic_timer_interrupt+0x6d/0x80
[ 1531.375036]  <EOI>
[ 1531.381333]  [<ffffffff81088cf1>] ? clockevents_program_event+0x71/0x100
[ 1531.403951]  [<ffffffff81033e26>] ? native_safe_halt+0x6/0x10
[ 1531.422830]  [<ffffffff810126b9>] default_idle+0x9/0x10
[ 1531.439994]  [<ffffffff81012e96>] arch_cpu_idle+0x26/0x30
[ 1531.457729]  [<ffffffff81081579>] cpu_startup_entry+0xe9/0x140
[ 1531.476894]  [<ffffffff8161c0e9>] start_secondary+0x1b1/0x1b4
[ 1531.495769] Code: 00 00 00 41 ff cc 75 e7 5b 41 5c 5d c3 66 66 66 66 66 2e 0f 1f 84 00 00 00 00 00 55 48 89 e5 5d c3 66 2e 0f 1f 84 00 00 00 00 00 <55> 48 89 e5 0f 31 89 c0 48 c1 e2 20 48 09 c2 48 89 d0 5d c3 66
[ 1531.558441] RIP  [<ffffffff810114e0>] hw_breakpoint_pmu_read+0x10/0x10
[ 1531.579902]  RSP <ffff88207fc43eb8>
[ 1531.591346] CR2: 0000000000000000
[ 1531.602222] ---[ end trace 4f01256e968f8dd2 ]---
[ 1531.617384] Kernel panic - not syncing: Fatal exception in interrupt
[ 1531.660010] Rebooting in 1 seconds..
[ 1532.670663] ACPI MEMORY or I/O RESET_REG.

Stack:
ffffffff81011529  ffff88207fc43eb8  <-  read_tsc+0x9
ffff88207fc43ee8  ffff88207fc43ec0  <-- rbp 
ffffffff8108247d  ffff88207fc43ec8  <-- ktime_get+0x4d
ffff88207fc4cb40  ffff88207fc43ed0
0000015357d3567c  ffff88207fc43ed8
0000000000000000  ffff88207fc43ee0
ffff88207fc43f10  ffff88207fc43ee8  <--- rbp 
ffffffff81088cbf  ffff88207fc44ef0  <--- clockevents_program_event+0x3f
ffff88207fc4d260  ffff88207fc44ef8
15263cfe0b480bb6  ffff88207fc44f00 
ffff88207fc4d160  ffff88207fc44f10
ffff88207fc43f20  ffff88207fc44f18  <---- rbp 
ffffffff8108a48f  ffff88207fc44f20  <---- tick_program_event+0x1f
ffff88207fc43f88 

(gdb) disassemble read_tsc+0x9
Dump of assembler code for function read_tsc:
   0xffffffff81011520 <+0>:     push   %rbp
   0xffffffff81011521 <+1>:     mov    %rsp,%rbp
   0xffffffff81011524 <+4>:     callq  *0xffffffff81a1fcb0
   0xffffffff8101152b <+11>:    mov    0xa040d6(%rip),%rdx        # 0xffffffff81a15608 <clocksource_tsc+8>
...
   0xffffffff81a1fcab <+235>:   81 ff ff ff ff e0       cmp    $0xe0ffffff,%edi
   0xffffffff81a1fcb1 <+241>:   14 01   adc    $0x1,%al
   0xffffffff81a1fcb3 <+243>:   81 ff ff ff ff f0       cmp    $0xf0ffffff,%edi
   0xffffffff81a1fcb9 <+249>:   3c 03   cmp    $0x3,%al
   0xffffffff81a1fcbb <+251>:   81 ff ff ff ff a0       cmp    $0xa0ffffff,%edi

*0xffffffff81a1fcb0 = 0xffffffff810114e0

(gdb) disassemble 0xffffffff810114e0
Dump of assembler code for function native_read_tsc:
   0xffffffff810114e0 <+0>:     push   %rbp
   0xffffffff810114e1 <+1>:     mov    %rsp,%rbp
   0xffffffff810114e4 <+4>:     rdtsc

The push instruction failed, ??? RSP (0018:ffff88207fc43eb8) is a valid address, because task.ti (0xffff881fe280a000)< RSP (0xffff88207fc43eb8) < task.ti + 8K (0xffff881fe280c000).

No conclusion because the two following reasons.
	1. unable to handle kernel NULL pointer dereference at (null)", and the "CR2: 0000000000000000", don't know why the NULL address comes from.
	2. the return address should be 0xffffffff8101152b, but in the top stack, we know is ffffffff81011529.


* case 2:

May 8 21:38:00 cdcftdp01 kernel: [1164489.346207] BUG: unable to handle kernel NULL pointer dereference at 0000000000000015
May 8 21:38:00 cdcftdp01 kernel: [1164489.372582] IP: [<ffffffff8129f552>] find_next_bit+0xa2/0xe0
May 8 21:38:00 cdcftdp01 kernel: [1164489.391794] PGD 3924dac067 PUD 39d43da067 PMD 0
May 8 21:38:00 cdcftdp01 kernel: [1164489.407589] Oops: 0000 1 SMP
May 8 21:38:00 cdcftdp01 kernel: [1164489.418789] Modules linked in: tscsync nf_conntrack_ipv6 nf_defrag_ipv6 ip6table_mangle igb_uio(O) ipt_MASQUERADE iptable_nat nf_nat_ipv4 nf_nat nf_conntrack_ipv4 nf_defrag_ipv4 xt_conntrack nf_conntrack ipt_REJECT xt_CHECKSUM iptable_mangle xt_tcpudp bridge stp llc ip6table_filter ip6_tables iptable_filter ip_tables ebtable_nat ebtables x_tables iTCO_wdt iTCO_vendor_support coretemp intel_rapl iosf_mbi kvm_intel kvm aesni_intel aes_x86_64 glue_helper lrw ablk_helper cryptd lpc_ich evdev acpi_power_meter hwmon ipv6
May 8 21:38:00 cdcftdp01 kernel: [1164489.570177] CPU: 35 PID: 42000 Comm: snort Tainted: G O 3.10.62-ltsi-WR6.0.0.29_standard #1
May 8 21:38:00 cdcftdp01 kernel: [1164489.601109] Hardware name: Cisco Systems Inc FPR4K-SM-44/FPR4K-SM-44, BIOS FXOSSM1.1.2.1.6.072020171212 07/20/2017
May 8 21:38:00 cdcftdp01 kernel: [1164489.635762] task: ffff883a7d2218c0 ti: ffff88373facc000 task.ti: ffff88373facc000
May 8 21:38:00 cdcftdp01 kernel: [1164489.660968] RIP: 0010:[<ffffffff8129f552>] [<ffffffff8129f552>] find_next_bit+0xa2/0xe0
May 8 21:38:00 cdcftdp01 kernel: [1164489.688196] RSP: 0018:ffff88373facd938 EFLAGS: 00010087
May 8 21:38:00 cdcftdp01 kernel: [1164489.706246] RAX: ffffffffffe00000 RBX: ffff883fdc52ad18 RCX: 0000000000000015
May 8 21:38:00 cdcftdp01 kernel: [1164489.730306] RDX: 0000000000000015 RSI: 0000000000000080 RDI: 0000000000000000
May 8 21:38:00 cdcftdp01 kernel: [1164489.754368] RBP: ffff88373facd938 R08: 0000000000000015 R09: 000000000000034d
May 8 21:38:00 cdcftdp01 kernel: [1164489.778430] R10: 0000000000000000 R11: 0000000000000000 R12: ffff8840799ad370
May 8 21:38:00 cdcftdp01 kernel: [1164489.802491] R13: 0000000000000000 R14: 0000000000000000 R15: 0000000000000001
May 8 21:38:00 cdcftdp01 kernel: [1164489.826552] FS: 00007fd2b3d34700(0000) GS:ffff8840799a0000(0000) knlGS:0000000000000000
May 8 21:38:00 cdcftdp01 kernel: [1164489.853764] CS: 0010 DS: 0000 ES: 0000 CR0: 0000000080050033
May 8 21:38:00 cdcftdp01 kernel: [1164489.873245] CR2: 0000000000000015 CR3: 00000039cec0e000 CR4: 00000000003407e0
May 8 21:38:00 cdcftdp01 kernel: [1164489.897305] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
May 8 21:38:00 cdcftdp01 kernel: [1164489.921367] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
May 8 21:38:00 cdcftdp01 kernel: [1164489.945426] Stack:
May 8 21:38:00 cdcftdp01 kernel: [1164489.952596] ffff88373facd948 ffffffff812908f9 ffff88373facdb10 0000000000000014
May 8 21:38:00 cdcftdp01 kernel: [1164489.977560] ffff88373facdaa0 ffffffff81072be3 01ff88373facda70 ffff88373facdb9c
May 8 21:38:00 cdcftdp01 kernel: [1164490.002523] fffffffffffffff8 00000000ffffffff 0000000000000057 0000000000000000
May 8 21:38:00 cdcftdp01 kernel: [1164490.027486] ffff883fdc52ad00 ffff883fdc52ad18 0000000000000000 0000000000000f9e
May 8 21:38:00 cdcftdp01 kernel: [1164490.052445] 0000000000000000 000000000000920a 000000000000000d 000000000000920a
May 8 21:38:00 cdcftdp01 kernel: [1164490.077411] 0000000000000000 0000000000000007 0000000000000000 0000000000000000
May 8 21:38:00 cdcftdp01 kernel: [1164490.102375] 0000000000000000 ffff883fdc52ad40 0000000000009158 000000000000b000
May 8 21:38:00 cdcftdp01 kernel: [1164490.127334] 0000000000000000 000000000000034d 0000000000009158 000000000000000d
May 8 21:38:00 cdcftdp01 kernel: [1164490.152298] 0000000000000001 000000000000001c 0000000000000000 0000000000000000
May 8 21:38:00 cdcftdp01 kernel: [1164490.177261] 0000000000000000 0000000000000000 0000000000000000 0000000000000000
May 8 21:38:00 cdcftdp01 kernel: [1164490.202219] ffff88373facdb9c 0000000106eea9dd ffff8840799b1c00 0000000000000023
May 8 21:38:00 cdcftdp01 kernel: [1164490.227181] ffff88373facdb10 ffff88373facdb88 ffffffff810735ae 0000002300000002
May 8 21:38:00 cdcftdp01 kernel: [1164490.252144] 0000000000011c00 ffff88373facdb9c ffff8840799ad370 ffff883fdc528480
May 8 21:38:00 cdcftdp01 kernel: [1164490.277106] 0000000000011c00 0000000000011c00 ffff8840799b1c00 ffff8840799b23d0
May 8 21:38:00 cdcftdp01 kernel: [1164490.302067] ffff883700000000 ffffffff81068db3 ffff883fdc52ad40 ffff883fdc528480
May 8 21:38:00 cdcftdp01 kernel: [1164490.327025] 0000000000000000 0000002300000000 ffff8840799b1c00 0000000000000000
May 8 21:38:00 cdcftdp01 kernel: [1164490.351987] 0000000200000000 0000000000000000 ffff8840799ad370 0000000000000000
May 8 21:38:00 cdcftdp01 kernel: [1164490.376944] 0000000000000020 ffff883fdc528480 0000000106eea9dd ffff8840799b1c00
May 8 21:38:00 cdcftdp01 kernel: [1164490.401904] 0000000000000023 0000000000000000 ffff88373facdbc8 ffffffff81073e1a
May 8 21:38:00 cdcftdp01 kernel: [1164490.426865] 0000000181068ead ffff883a7d221bb8 ffff8840799b1c00 0000000000000023
May 8 21:38:00 cdcftdp01 kernel: [1164490.451829] ffff883a7d2218c0 ffff88373facddc0 ffff88373facdcd0 ffffffff8162892c
May 8 21:38:00 cdcftdp01 kernel: [1164490.476789] ffff88373facdfd8 0000000000011c00 ffff88373facdfd8 0000000000011c00
May 8 21:38:00 cdcftdp01 kernel: [1164490.501757] ffff883a7d2218c0 0000000000000000 ffff88373facdd60 0000000000000001
May 8 21:38:00 cdcftdp01 kernel: [1164490.526720] ffff8840799ad1e0 0000000000000001 ffff88373facdce0 0000000000000282
May 8 21:38:00 cdcftdp01 kernel: [1164490.551679] ffff88373facdc58 ffffffff81297060 ffff8840799ad1e0 ffff88373facdd60
May 8 21:38:00 cdcftdp01 kernel: [1164490.576641] ffff88373facdc78 ffffffff810634c2 0000000000000000 0000000000000282
May 8 21:38:00 cdcftdp01 kernel: [1164490.601606] ffff88373facdcd0 ffffffff81063a95 152cd976f48a1958 0000000000000001
May 8 21:38:00 cdcftdp01 kernel: [1164490.626568] 1528b768f19888f4 0000000000000282 ffff88373facdda8 ffff88373facdd60
May 8 21:38:00 cdcftdp01 kernel: [1164490.651528] ffffffff81bcbc38 ffff883a7d2218c0 ffff88373facddc0 ffff88373facdce0
May 8 21:38:00 cdcftdp01 kernel: [1164490.676490] ffffffff816289d4 ffff88373facdd20 ffffffff8108ae1c 0000000000000000
May 8 21:38:00 cdcftdp01 kernel: [1164490.701448] ffff88373facdda8 ffff88373facdd60 0000000000000000 0000000000831199
May 8 21:38:00 cdcftdp01 kernel: [1164490.726413] ffff883a7d2218c0 ffff88373facde40 ffffffff8108bb95 ffffffff00000000
May 8 21:38:00 cdcftdp01 kernel: [1164490.751377] Call Trace:
May 8 21:38:00 cdcftdp01 kernel: [1164490.759983] [<ffffffff812908f9>] __next_cpu+0x19/0x30
May 8 21:38:00 cdcftdp01 kernel: [1164490.777464] [<ffffffff81072be3>] ? find_busiest_group+0x123/0xa10
May 8 21:38:00 cdcftdp01 kernel: [1164490.798377] [<ffffffff810735ae>] ? load_balance+0xde/0x6d0
May 8 21:38:00 cdcftdp01 kernel: [1164490.817289] [<ffffffff81068db3>] ? update_rq_clock.part.58+0x13/0x30
May 8 21:38:00 cdcftdp01 kernel: [1164490.839061] [<ffffffff81073e1a>] ? idle_balance+0xca/0x130
May 8 21:38:00 cdcftdp01 kernel: [1164490.857974] [<ffffffff8162892c>] ? __schedule+0x77c/0x800
May 8 21:38:00 cdcftdp01 kernel: [1164490.876600] [<ffffffff81297060>] ? timerqueue_add+0x60/0xb0
May 8 21:38:00 cdcftdp01 kernel: [1164490.895797] [<ffffffff810634c2>] ? enqueue_hrtimer+0x22/0x50
May 8 21:38:00 cdcftdp01 kernel: [1164490.915278] [<ffffffff81063a95>] ? __hrtimer_start_range_ns+0xf5/0x250
May 8 21:38:00 cdcftdp01 kernel: [1164490.937625] [<ffffffff816289d4>] ? schedule+0x24/0x60
May 8 21:38:00 cdcftdp01 kernel: [1164490.955108] [<ffffffff8108ae1c>] ? futex_wait_queue_me+0xcc/0x100
May 8 21:38:00 cdcftdp01 kernel: [1164490.976024] [<ffffffff8108bb95>] ? futex_wait+0x165/0x250
May 8 21:38:00 cdcftdp01 kernel: [1164490.994648] [<ffffffff81063630>] ? hrtimer_get_res+0x40/0x40
May 8 21:38:00 cdcftdp01 kernel: [1164491.014131] [<ffffffff8108d24d>] ? do_futex+0xdd/0x9e0
May 8 21:38:00 cdcftdp01 kernel: [1164491.031897] [<ffffffff816284ae>] ? __schedule+0x2fe/0x800
May 8 21:38:00 cdcftdp01 kernel: [1164491.050523] [<ffffffff8108dbc7>] ? SyS_futex+0x77/0x150
May 8 21:38:00 cdcftdp01 kernel: [1164491.068578] [<ffffffff81630209>] ? system_call_fastpath+0x16/0x1b
May 8 21:38:00 cdcftdp01 kernel: [1164491.089490] Code: c7 c2 ff ff ff ff 29 f1 48 d3 ea 48 21 d0 74 41 f3 48 0f bc c0 48 01 f8 5d c3 0f 1f 80 00 00 00 00 48 c7 c0 ff ff ff ff 48 d3 e0 <49> 23 00 48 83 fe 3f 76 c6 48 85 c0 75 d7 49 83 c0 08 48 83 ee
May 8 21:38:00 cdcftdp01 kernel: [1164491.152899] RIP [<ffffffff8129f552>] find_next_bit+0xa2/0xe0
May 8 21:38:00 cdcftdp01 kernel: [1164491.172398] RSP <ffff88373facd938>
May 8 21:38:00 cdcftdp01 kernel: [1164491.184433] CR2: 0000000000000015


(gdb) disassemble find_next_bit
Dump of assembler code for function find_next_bit:
   0xffffffff8129f4b0 <+0>:     push   %rbp
   0xffffffff8129f4b1 <+1>:     cmp    %rsi,%rdx
   0xffffffff8129f4b4 <+4>:     mov    %rsi,%rax
...
   0xffffffff8129f54f <+159>:   shl    %cl,%rax
   0xffffffff8129f552 <+162>:   and    (%r8),%rax  <======= r8 is a illegal address
   0xffffffff8129f555 <+165>:   cmp    $0x3f,%rsi

(gdb) disassemble __next_cpu
Dump of assembler code for function __next_cpu:
   0xffffffff812908e0 <+0>:     push   %rbp
   0xffffffff812908e1 <+1>:     mov    %rsi,%rax
   0xffffffff812908e4 <+4>:     inc    %edi
   0xffffffff812908e6 <+6>:     movslq %edi,%rdx
   0xffffffff812908e9 <+9>:     mov    $0x80,%esi
   0xffffffff812908ee <+14>:    mov    %rax,%rdi
   0xffffffff812908f1 <+17>:    mov    %rsp,%rbp
   0xffffffff812908f4 <+20>:    callq  0xffffffff8129f4b0 <find_next_bit>
   0xffffffff812908f9 <+25>:    mov    $0x80,%edx


Stack:
ffff88373facd948   RSP: ffff88373facd938  RBP: ffff88373facd938 <-  rbp
ffffffff812908f9	ffff88373facd940  \_\_next_cpu+0x19
ffff88373facdb10	ffff88373facd948  	                <-- rbp
0000000000000014	ffff88373facd950   ??
ffff88373facdaa0 
ffffffff81072be3 
01ff88373facda70 

Guess:
The address of 0000000000000014 is out of kernel text segment. Stack corruption occurs.
R8 is a illegal address, still don't know why it happens.


static int __init code_bytes_setup(char *s) 
{
        ssize_t ret;
        unsigned long val;

        if (!s)
                return -EINVAL;

        ret = kstrtoul(s, 0, &val);
        if (ret)
                return ret;

        code_bytes = val;
        if (code_bytes > 8192)
                code_bytes = 8192;

        return 1;
}
__setup("code_bytes=", code_bytes_setup);
