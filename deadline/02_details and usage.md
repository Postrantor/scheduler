---
tip: translate by baidu@2023-09-05 16:03:39
url: https://lwn.net/Articles/743946/
title: Deadline scheduler part 2 — details and usage [LWN.net]
date: 2023-09-05 16:00:38
tag: 
summary: Linux’s deadline scheduler is a global early deadline first scheduler for sporadic tasks with constra......
> 摘要：Linux的截止日期调度程序是
---

Linux’s deadline scheduler is a global early deadline first scheduler for sporadic tasks with constrained deadlines. These terms were defined in [the first part of this series](https://lwn.net/Articles/743740/). In this installment, the details of the Linux deadline scheduler and how it can be used will be examined.

> Linux 的截止日期调度程序是一个全局提前截止日期优先调度程序，用于具有受限截止日期的零星任务。这些术语在[本系列的第一部分]中进行了定义(https://lwn.net/Articles/743740/)。在本期文章中，将研究Linux截止日期调度程序的详细信息以及如何使用它。

The deadline scheduler prioritizes the tasks according to the task’s job deadline: the earliest absolute deadline first. For a system with _M_ processors, the _M_ earliest deadline jobs will be selected to run on the _M_ processors.

> 截止日期调度器根据任务的作业截止日期对任务进行优先级排序：首先是最早的绝对截止日期。对于具有*M*处理器的系统，将选择在*M*处理器上运行的*M*最早截止日期作业。

The Linux deadline scheduler also implements the constant bandwidth server (CBS) algorithm, which is a resource-reservation protocol. CBS is used to guarantee that each task will receive its full run time during every period. At every activation of a task, the CBS replenishes the task’s run time. As the job runs, it consumes that time; if the task runs out, it will be throttled and descheduled. In this case, the task will be able to run only after the next replenishment at the beginning of the next period. Therefore, CBS is used to both guarantee each task’s CPU time based on its timing requirements and to prevent a misbehaving task from running for more than its run time and causing problems to other jobs.

> Linux 截止日期调度器还实现了恒定带宽服务器（CBS）算法，这是一种资源预留协议。CBS 用于保证每个任务在每个时段都能获得完整的运行时间。每次激活任务时，CBS 都会补充任务的运行时间。当作业运行时，它会消耗时间；如果任务用完，它将被限制并取消计划。在这种情况下，只有在下一个周期开始的下一次补货之后，任务才能运行。因此，CBS 用于根据每个任务的时间要求保证其 CPU 时间，并防止行为不端的任务运行时间超过其运行时间并给其他作业带来问题。

In order to avoid overloading the system with deadline tasks, the deadline scheduler implements an acceptance test, which is done every time a task is configured to run with the deadline scheduler. This test guarantees that deadline tasks will not use more than the maximum amount of the system's CPU time, which is specified using the kernel.sched_rt_runtime_us and kernel.sched_rt_period_us sysctl knobs. The default values are 950000 and 1000000, respectively, limiting realtime tasks to 950,000µs of CPU time every 1s of wall-clock time. For a single-core system, this test is both necessary and sufficient. It means that the acceptance of a task guarantees that the task will be able to use all the run time allocated to it before its deadline.

> 为了避免最后期限任务使系统过载，最后期限调度器实现了验收测试，每次任务配置为使用最后期限调度器运行时都会进行验收测试。此测试确保截止日期任务使用的 CPU 时间不会超过系统的最大时间，该时间是使用 kernel.sched_rt_runtime_us 和 kernel.sached_rt_rperiod_us sysctl 旋钮指定的。默认值分别为 950000 和 1000000，将实时任务限制为每 1 秒墙上时钟的 CPU 时间为 950000µs。对于单核心系统，此测试既是必要的，也是充分的。这意味着接受任务可以保证任务能够在截止日期前使用分配给它的所有运行时间。

However, it is worth noting that this acceptance test is necessary, but _not_ sufficient, for global scheduling on multiprocessor systems. As Dhall’s effect (described in the first part of this series) shows, the global deadline scheduler acceptance task is unable to schedule the task set even though there is CPU time available. Hence, the current acceptance test does not guarantee that, once accepted, the tasks will be able to use all the assigned run time before their deadlines. The best the current acceptance task can guarantee is bounded tardiness, which is a good guarantee for soft real-time systems. If the user wants to guarantee that all tasks will meet their deadlines, the user has to either use a partitioned approach or to use a necessary and sufficient acceptance test, defined by:

> 然而，值得注意的是，对于多处理器系统上的全局调度，这种验收测试是必要的，但并不充分。正如 Dhall 的效果（在本系列的第一部分中描述）所示，即使有可用的 CPU 时间，全局截止日期调度器接受任务也无法调度任务集。因此，当前的验收测试不能保证一旦被接受，任务将能够在截止日期之前使用所有分配的运行时间。当前验收任务所能保证的最好的是有界延迟，这对软实时系统来说是一个很好的保证。如果用户希望保证所有任务都能在截止日期前完成，则用户必须使用分区方法或使用必要且充分的验收测试，定义如下：

Σ(WCETi / Pi) <= M - (M - 1) x Umax

Or, expressed in words: the sum of the run time/period of each task should be less than or equal to the number of processors, minus the largest utilization multiplied by the number of processors minus one. It turns out that, the bigger Umax is, the less load the system is able to handle.

> 或者，用文字表示：每个任务的运行时间/周期的总和应该小于或等于处理器的数量，减去最大利用率乘以处理器的数量减去一。事实证明，Umax 越大，系统能够处理的负载就越少。

In the presence of tasks with a big utilization, one good strategy is to partition the system and isolate some high-load tasks in a way that allows the small-utilization tasks to be globally scheduled on a different set of CPUs. Currently, the deadline scheduler does not enable the user to set the affinity of a thread, but it is possible to partition a system using control-group cpusets.

> 在存在利用率高的任务的情况下，一个好的策略是对系统进行分区，并隔离一些高负载任务，从而允许在不同的 CPU 集上全局调度利用率低的任务。目前，截止日期调度程序不允许用户设置线程的相关性，但可以使用控制组 cpusets 对系统进行分区。

For example, consider a system with eight CPUs. One big task has a utilization close to 90% of one CPU, while a set of many other tasks have a lower utilization. In this environment, one recommended setup would be to isolate CPU0 to run the high-utilization task while allowing the other tasks to run in the remaining CPUs. To configure this environment, the user must follow the following steps:

> 例如，考虑一个有八个 CPU 的系统。一个大任务的利用率接近一个 CPU 的 90%，而许多其他任务的利用度较低。在这种环境中，一种建议的设置是隔离 CPU0 以运行高利用率任务，同时允许其他任务在剩余的 CPU 中运行。要配置此环境，用户必须遵循以下步骤：

1.  Enter in the cpuset directory and create two cpusets:

    ```
        # cd /sys/fs/cgroup/cpuset/
        # mkdir cluster
        # mkdir partition


    ```

2.  Disable load balancing in the root cpuset to create two new root domains in the CPU sets:

> 2.在根 cpuset 中禁用负载平衡，以便在 CPU 集中创建两个新的根域：

    ```
        # echo 0 > cpuset.sched_load_balance


    ```

3.  Enter the directory for the cluster cpuset, set the CPUs available to 1-7, the memory node the set should run in (in this case the system is not NUMA, so it is always node zero), and set the cpuset to the exclusive mode.

> 3.输入集群 cpuset 的目录，将可用的 CPU 设置为 1-7，设置应该在其中运行的内存节点（在这种情况下，系统不是 NUMA，所以它总是节点零），并将 cpuset 设置为独占模式。

    ```
        # cd cluster/
        # echo 1-7 > cpuset.cpus
        # echo 0 > cpuset.mems
        # echo 1 > cpuset.cpu_exclusive


    ```

4.  Move all tasks to this CPU set

    ```
        # ps -eLo lwp | while read thread; do echo $thread > tasks ; done


    ```

5.  Then it is possible to start deadline tasks in this cpuset.
6.  Configure the partition cpuset:

    ```
        # cd ../partition/
        # echo 1 > cpuset.cpu_exclusive
        # echo 0 > cpuset.mems
        # echo 0 > cpuset.cpus


    ```

7.  Finally move the shell to the partition cpuset.

    ```
        # echo $$ > tasks


    ```

8.  The final step is to run the deadline workload.

With this setup, the task isolated in the partitioned cpuset will not interfere with the tasks in the cluster cpuset, increasing the system’s maximum load while meeting the deadline of real-time tasks.

> 通过此设置，在分区 cpuset 中隔离的任务不会干扰集群 cpuset 的任务，从而在满足实时任务的截止日期的同时增加系统的最大负载。

#### The developer’s perspective

There are three ways to use the deadline scheduler: as constant bandwidth server, as a periodic/sporadic server waiting for an event, or with a periodic task waiting for replenishment. The most basic parameter for the sched deadline is the period, which defines how often a task is activated. When a task does not have an activation pattern, it is possible to use the deadline scheduler in an aperiodic mode by using only the CBS features.

> 有三种方法可以使用截止日期调度程序：作为恒定带宽服务器，作为等待事件的周期性/偶发性服务器，或者使用等待补充的周期性任务。sched 截止日期最基本的参数是 period，它定义了任务被激活的频率。当任务没有激活模式时，可以通过仅使用 CBS 功能以非周期模式使用截止日期调度器。

In the aperiodic case, the best thing the user can do is to estimate how much CPU time a task needs in a given period of time to accomplish the expected result. For instance, if one task needs 200ms each second to accomplish its work, run time would be 200,000,000ns and the period would be 1,000,000,000ns. The [sched_setattr()](http://man7.org/linux/man-pages/man2/sched_setattr.2.html) system call is used to set the deadline-scheduling parameters. The following code is a simple example of how to set the mentioned parameters in an application:

> 在非周期性的情况下，用户能做的最好的事情就是估计任务在给定的时间段内需要多少 CPU 时间来实现预期结果。例如，如果一个任务每秒需要 200 毫秒来完成其工作，则运行时间将为 200000000ns，周期将为 1000000000ns。[sched_setattr（）](http://man7.org/linux/man-pages/man2/sched_setattr.2.html)系统调用用于设置截止日期调度参数。以下代码是如何在应用程序中设置上述参数的简单示例：

```
    int main (int argc, char **argv)
    {
        int ret;
        int flags = 0;
        struct sched_attr attr;

        memset(&attr, 0, sizeof(attr));
        attr.size = sizeof(attr);

        /* This creates a 200ms / 1s reservation */
        attr.sched_policy   = SCHED_DEADLINE;
        attr.sched_runtime  =  200000000;
        attr.sched_deadline = attr.sched_period = 1000000000;

        ret = sched_setattr(0, &attr, flags);
        if (ret < 0) {
            perror("sched_setattr failed to set the priorities");
            exit(-1);
        }

        do_the_computation_without_blocking();
        exit(0);
}


```

In the aperiodic case, the task does not need to know when a period starts, and so the task just needs to run, knowing that the scheduler will throttle the task after it has consumed the specified run time.

> 在非周期性的情况下，任务不需要知道某个周期何时开始，因此任务只需要运行，因为它知道调度程序会在任务消耗指定的运行时间后对其进行节流。

Another use case is to implement a periodic task which starts to run at every periodic run-time replenishment, runs until it finishes its processing, then goes to sleep until the next activation. Using the parameters from the previous example, the following code sample uses the [sched_yield()](http://man7.org/linux/man-pages/man2/sched_yield.2.html) system call to notify the scheduler of end of the current activation. The task will be awakened by the next run-time replenishment. Note that the semantics of sched_yield() are a bit different for deadline tasks; they will not be scheduled again until the run-time replenishment happens.

> 另一个用例是实现一个周期性任务，该任务在每次周期性的运行时补充时开始运行，运行到完成处理，然后进入睡眠状态，直到下一次激活。使用上一个示例中的参数，下面的代码示例使用[sched_yield（）](http://man7.org/linux/man-pages/man2/sched_yield.2.html)系统调用以通知调度程序当前激活结束。该任务将被下一次运行时补给唤醒。请注意，对于截止日期任务，sched_yield（）的语义有点不同；在运行时补充发生之前，不会再次安排它们。

Code working in this mode would look like the example above, except that the actual computation looks like:

> 在这种模式下工作的代码看起来像上面的例子，只是实际计算看起来像：

```
        for(;;) {
            do_the_computation();
            /*
	     * Notify the scheduler the end of the computation
             * This syscall will block until the next replenishment
             */
	    sched_yield();
        }


```

It is worth noting that the computation must finish within the given run time. If the task does not finish, it will be throttled by the CBS algorithm.

> 值得注意的是，计算必须在给定的运行时间内完成。如果任务没有完成，它将被 CBS 算法抑制。

The most common realtime use case for the realtime task is to wait for an external event to take place. In this case, the task waits in a blocking system call. This system call will wake up the real-time task with, at least, a minimum interval between each activation. That is, it is a sporadic task. Once activated, the task will do the computation and provide the response. Once the task provides the output, the task goes to sleep by blocking waiting for the next event.

> 实时任务最常见的实时用例是等待外部事件发生。在这种情况下，任务在阻塞系统调用中等待。该系统调用将唤醒实时任务，每次激活之间至少有一个最小间隔。也就是说，这是一项零星的任务。一旦激活，任务将进行计算并提供响应。一旦任务提供了输出，该任务就会通过阻塞等待下一个事件而进入睡眠状态。

```
        for(;;) {
            /*
	     * Block in a blocking system call waiting for a data
             * to be processed.
             */
            process_the_data();
            produce_the_result()
	    block_waiting_for_the_next_event();
        }


```

#### Conclusion

The deadline scheduler is able to provide guarantees for realtime tasks based only in the task’s timing constraints. Although global multi-core scheduling faces Dhall’s effect, it is possible to configure the system to achieve a high load utilization using cpusets as a method to partition the systems. Developers can also benefit from the deadline scheduler by designing their application to interact with the scheduler, simplifying the control of the timing behavior of the task.

> 截止日期调度器能够仅基于任务的时间约束来为实时任务提供保证。尽管全局多核调度面临着 Dhall 的影响，但使用 cpusets 作为系统分区的方法来配置系统以实现高负载利用率是可能的。开发人员还可以通过设计他们的应用程序来与调度器交互，从而简化对任务定时行为的控制，从而从截止日期调度器中受益。

The deadline scheduler tasks have a higher priority than realtime scheduler tasks. That means that even the highest fixed-priority task will be delayed by deadline tasks. Thus, deadline tasks do not need to consider interference from realtime tasks, but realtime tasks must consider interference from deadline tasks.

> 截止日期调度程序任务的优先级高于实时调度程序任务。这意味着，即使是固定优先级最高的任务也会因截止日期任务而延迟。因此，截止日期任务不需要考虑来自实时任务的干扰，但实时任务必须考虑来自截止日期任务的干扰。

The deadline scheduler and the PREEMPT_RT patch play different roles in improving Linux’s realtime features. While the deadline scheduler allows scheduling tasks in a more predictable way, the PREEMPT_RT patch set improves the kernel by reducing and limiting the amount of time a lower-priority task can delay the execution of a realtime task. It works by reducing the amount of the time a processor runs with preemption and IRQs disabled and the amount of time in which a lower-priority task can delay the execution of a task by holding a lock.

> 截止日期调度程序和 PREEMPT_RT 补丁在改进 Linux 的实时功能方面发挥着不同的作用。虽然截止日期调度器允许以更可预测的方式调度任务，但 PREEMPT_RT 补丁集通过减少和限制优先级较低的任务可能延迟实时任务执行的时间来改进内核。它的工作原理是减少处理器在抢占和 IRQ 被禁用的情况下运行的时间量，以及优先级较低的任务可以通过持有锁来延迟任务执行的时间量。

For example, as a realtime task can suffer an activation latency higher than 5ms when running in a non-realtime kernel, it is that this kernel cannot handle deadline tasks with deadlines shorter than 5ms. In contrast, the realtime kernel provides a guarantee, on well tuned and certified hardware, of not delaying the start of the highest priority task by more than 150µs, thus it is possible to handle realtime tasks with deadlines much shorter than 5ms. You can find more about the realtime kernel [here](http://developerblog.redhat.com/?p=425603&preview_id=425603&preview_nonce=28c03def3d&post_format=standard&preview=true).

> 例如，由于实时任务在非实时内核中运行时可能会遭受高于 5ms 的激活延迟，因此该内核无法处理截止日期小于 5ms 的截止日期任务。相比之下，实时内核在经过良好调整和认证的硬件上提供了一种保证，即最高优先级任务的启动不会延迟 150µs 以上，因此可以处理截止日期远短于 5ms 的实时任务。你可以在这里找到更多关于实时内核的信息(http://developerblog.redhat.com/?p=425603&preview_id=425603&preview_nonce=28c03def3d&post_format=standard&preview=true)。

Acknowledgment: this series of articles was reviewed and improved with comments from Clark Williams, Beth Uptagrafft, Arnaldo Carvalho de Melo, Luis Claudio R. Gonçalves, Oleksandr Natalenko, Jiri Kastner and Tommaso Cucinotta.

> 鸣谢：本系列文章由 Clark Williams、Beth Uptagrafft、Arnaldo Carvalho de Melo、Luis Claudio R.Gonçalves、Oleksandr Natalenko、Jiri Kastner 和 Tommaso Cucinotta 进行了评论和改进。

<table><tbody><tr><th colspan="2">Index entries for this article</th></tr><tr><td><a href="https://lwn.net/Kernel/Index">Kernel</a></td><td><a href="https://lwn.net/Kernel/Index#Realtime-Deadline_scheduling">Realtime/Deadline scheduling</a></td></tr><tr><td><a href="https://lwn.net/Kernel/Index">Kernel</a></td><td><a href="https://lwn.net/Kernel/Index#Scheduler-Deadline_scheduling">Scheduler/Deadline scheduling</a></td></tr><tr><td><a href="https://lwn.net/Archives/GuestIndex/">GuestArticles</a></td><td><a href="https://lwn.net/Archives/GuestIndex/#Bristot_de_Oliveira_Daniel">Bristot de Oliveira, Daniel</a></td></tr></tbody></table>

([Log in](https://lwn.net/Login/?target=/Articles/743946/) to post comments)

> （[登录](https://lwn.net/Login/?target=/Articles/743946/)发布评论）
