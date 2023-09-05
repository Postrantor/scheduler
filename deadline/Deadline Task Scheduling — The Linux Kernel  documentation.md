---
tip: translate by baidu@2023-09-05 16:11:49
url: https://docs.kernel.org/scheduler/sched-deadline.html
title: Deadline Task Scheduling — The Linux Kernel  documentation
date: 2023-09-05 16:10:40
tag: 

summary: Fiddling with these settings can result in an unpredictable or even unstable

> 总结：篡改这些设置可能会导致不可预测甚至不稳定
system behavior. As for ......
---

## 0. WARNING[¶](#warning "Permalink to this heading")

Fiddling with these settings can result in an unpredictable or even unstable system behavior. As for -rt (group) scheduling, it is assumed that root users know what they're doing.

> 篡改这些设置可能会导致不可预测甚至不稳定的系统行为。至于-rt（group）调度，假设根用户知道他们在做什么。

## 1. Overview[¶](#overview "Permalink to this heading")

The SCHED_DEADLINE policy contained inside the sched_dl scheduling class is basically an implementation of the Earliest Deadline First (EDF) scheduling algorithm, augmented with a mechanism (called Constant Bandwidth Server, CBS) that makes it possible to isolate the behavior of tasks between each other.

> SCHED_dl 调度类中包含的 SCHED_DEADLINE 策略基本上是最早截止日期优先（EDF）调度算法的实现，并添加了一种机制（称为恒定带宽服务器，CBS），可以将任务的行为隔离在彼此之间。

## 2. Scheduling algorithm[¶](#scheduling-algorithm "Permalink to this heading")

### 2.1 Main algorithm[¶](#main-algorithm "Permalink to this heading")

SCHED_DEADLINE [18] uses three parameters, named "runtime", "period", and "deadline", to schedule tasks. A SCHED_DEADLINE task should receive "runtime" microseconds of execution time every "period" microseconds, and these "runtime" microseconds are available within "deadline" microseconds from the beginning of the period. In order to implement this behavior, every time the task wakes up, the scheduler computes a "scheduling deadline" consistent with the guarantee (using the CBS[2,3] algorithm). Tasks are then scheduled using EDF[1] on these scheduling deadlines (the task with the earliest scheduling deadline is selected for execution). Notice that the task actually receives "runtime" time units within "deadline" if a proper "admission control" strategy (see Section "4. Bandwidth management") is used (clearly, if the system is overloaded this guarantee cannot be respected).

> SCHED_DEADLINE[18]使用三个参数，分别命名为“运行时”、“周期”和“截止日期”来调度任务。SCHED_DEADLINE 任务应每“周期”微秒接收“运行时”微秒的执行时间，并且这些“运行时间”微秒在周期开始后的“截止日期”微秒内可用。为了实现这种行为，每次任务唤醒时，调度器都会计算一个与保证一致的“调度截止日期”（使用 CBS[2,3]算法）。然后使用 EDF[1]在这些调度截止日期对任务进行调度（选择调度截止日期最早的任务执行）。请注意，如果采用适当的“准入控制”策略，任务实际上会在“截止日期”内接收“运行时”时间单位（请参见第 4 节）。带宽管理”）（显然，如果系统过载，则不能遵守此保证）。

Summing up, the CBS[2,3] algorithm assigns scheduling deadlines to tasks so that each task runs for at most its runtime every period, avoiding any interference between different tasks (bandwidth isolation), while the EDF[1] algorithm selects the task with the earliest scheduling deadline as the one to be executed next. Thanks to this feature, tasks that do not strictly comply with the "traditional" real-time task model (see Section 3) can effectively use the new policy.

> 总之，CBS[2,3]算法为任务分配调度截止日期，以便每个任务在每个周期最多运行其运行时间，避免不同任务之间的任何干扰（带宽隔离），而 EDF[1]算法则选择调度截止日期最早的任务作为下一个要执行的任务。得益于此功能，不严格遵守“传统”实时任务模型（见第 3 节）的任务可以有效地使用新策略。

In more details, the CBS algorithm assigns scheduling deadlines to tasks in the following way:

> 更详细地说，CBS 算法以以下方式为任务分配调度截止日期：

- Each SCHED_DEADLINE task is characterized by the "runtime", "deadline", and "period" parameters;

> \*每个 SCHED_DEADLINE 任务都具有“运行时”、“截止日期”和“周期”参数的特征；

- The state of the task is described by a "scheduling deadline", and a "remaining runtime". These two parameters are initially set to 0;

> \*任务的状态由“调度截止日期”和“剩余运行时间”来描述。这两个参数最初设置为 0；

- When a SCHED_DEADLINE task wakes up (becomes ready for execution), the scheduler checks if:

> \*当 SCHED_DEADLINE 任务唤醒（准备执行）时，调度程序会检查是否：

    ```
             remaining runtime                  runtime
    ----------------------------------    >    ---------
    scheduling deadline - current time           period


    ```

    then, if the scheduling deadline is smaller than the current time, or this condition is verified, the scheduling deadline and the remaining runtime are re-initialized as

    scheduling deadline = current time + deadline remaining runtime = runtime

    otherwise, the scheduling deadline and the remaining runtime are left unchanged;

- When a SCHED_DEADLINE task executes for an amount of time t, its remaining runtime is decreased as:

> \*当 SCHED_DEADLINE 任务执行一段时间 t 时，其剩余运行时间将减少为：

    ```
    remaining runtime = remaining runtime - t


    ```

    (technically, the runtime is decreased at every tick, or when the task is descheduled / preempted);

- When the remaining runtime becomes less or equal than 0, the task is said to be "throttled" (also known as "depleted" in real-time literature) and cannot be scheduled until its scheduling deadline. The "replenishment time" for this task (see next item) is set to be equal to the current value of the scheduling deadline;

> \*当剩余运行时间小于或等于 0 时，任务被称为“节流”（在实时文献中也称为“耗尽”），并且在其调度截止日期之前无法进行调度。此任务的“补货时间”（见下一项）设置为等于计划截止日期的当前值；

- When the current time is equal to the replenishment time of a throttled task, the scheduling deadline and the remaining runtime are updated as:

> \*当当前时间等于限制任务的补充时间时，调度截止日期和剩余运行时间更新为：

    ```
    scheduling deadline = scheduling deadline + period
    remaining runtime = remaining runtime + runtime


    ```

The SCHED_FLAG_DL_OVERRUN flag in sched_attr's sched_flags field allows a task to get informed about runtime overruns through the delivery of SIGXCPU signals.

> SCHED_attr 的 SCHED_flags 字段中的 SCHED_FLAG_DL_OVERRUN 标志允许任务通过传递 SIGXCPU 信号来获得有关运行时溢出的信息。

### 2.2 Bandwidth reclaiming[¶](#bandwidth-reclaiming "Permalink to this heading")

Bandwidth reclaiming for deadline tasks is based on the GRUB (Greedy Reclamation of Unused Bandwidth) algorithm [15, 16, 17] and it is enabled when flag SCHED_FLAG_RECLAIM is set.

> 截止日期任务的带宽回收基于 GRUB（未使用带宽的贪婪回收）算法[15，16，17]，并且在设置标志 SCHED_flag_RECLAIM 时启用。

The following diagram illustrates the state names for tasks handled by GRUB:

> 下图说明了 GRUB 处理的任务的状态名称：

```
                    ------------
        (d)        |   Active   |
     ------------->|            |
     |             | Contending |
     |              ------------
     |                A      |
 ----------           |      |
|          |          |      |
| Inactive |          |(b)   | (a)
|          |          |      |
 ----------           |      |
     A                |      V
     |              ------------
     |             |   Active   |
     --------------|     Non    |
        (c)        | Contending |
                    ------------


```

A task can be in one of the following states:

- ActiveContending: if it is ready for execution (or executing);

- ActiveNonContending: if it just blocked and has not yet surpassed the 0-lag time;

> \*ActiveNonContending：如果它刚刚被阻止，并且还没有超过 0 滞后时间；

- Inactive: if it is blocked and has surpassed the 0-lag time.

State transitions:

1.  When a task blocks, it does not become immediately inactive since its bandwidth cannot be immediately reclaimed without breaking the real-time guarantees. It therefore enters a transitional state called ActiveNonContending. The scheduler arms the "inactive timer" to fire at the 0-lag time, when the task's bandwidth can be reclaimed without breaking the real-time guarantees.

> 1.当任务阻塞时，它不会立即变为非活动状态，因为它的带宽不能在不破坏实时保证的情况下立即回收。因此，它进入一种称为 ActiveNonContending 的过渡状态。调度程序将“非活动计时器”设置为在 0 滞后时间启动，此时可以在不破坏实时保证的情况下回收任务的带宽。

    The 0-lag time for a task entering the ActiveNonContending state is computed as:

    ```
               (runtime * dl_period)
    deadline - ---------------------
                    dl_runtime


    ```

    where runtime is the remaining runtime, while dl_runtime and dl_period are the reservation parameters.

2.  If the task wakes up before the inactive timer fires, the task re-enters the ActiveContending state and the "inactive timer" is canceled. In addition, if the task wakes up on a different runqueue, then the task's utilization must be removed from the previous runqueue's active utilization and must be added to the new runqueue's active utilization. In order to avoid races between a task waking up on a runqueue while the "inactive timer" is running on a different CPU, the "dl_non_contending" flag is used to indicate that a task is not on a runqueue but is active (so, the flag is set when the task blocks and is cleared when the "inactive timer" fires or when the task wakes up).

> 2.如果任务在非活动计时器触发之前唤醒，则任务将重新进入 ActiveContending 状态，并取消“非活动计时器”。此外，如果任务在不同的运行队列上唤醒，则必须从以前的运行队列的活动利用率中删除该任务的利用率，并将其添加到新的运行队列中。为了避免当“非活动计时器”在不同的 CPU 上运行时，在运行队列上唤醒的任务之间发生竞争，“dl_non_contending”标志用于指示任务不在运行队列中，但处于活动状态（因此，当任务阻塞时设置该标志，当“非激活计时器”启动或任务唤醒时清除该标志）。

3.  When the "inactive timer" fires, the task enters the Inactive state and its utilization is removed from the runqueue's active utilization.

> 3.当“非活动计时器”触发时，任务将进入非活动状态，其利用率将从运行队列的活动利用率中删除。

4.  When an inactive task wakes up, it enters the ActiveContending state and its utilization is added to the active utilization of the runqueue where it has been enqueued.

> 4.当非活动任务唤醒时，它进入 ActiveContending 状态，并且它的利用率被添加到它已入队的运行队列的活动利用率中。

For each runqueue, the algorithm GRUB keeps track of two different bandwidths:

> 对于每个运行队列，算法 GRUB 跟踪两个不同的带宽：

- Active bandwidth (running_bw): this is the sum of the bandwidths of all tasks in active state (i.e., ActiveContending or ActiveNonContending);

> \*活动带宽（running_bw）：这是所有处于活动状态的任务（即 ActiveContending 或 ActiveNonContending）的带宽之和；

- Total bandwidth (this_bw): this is the sum of all tasks "belonging" to the runqueue, including the tasks in Inactive state.

> \*总带宽（this_bw）：这是“属于”运行队列的所有任务的总和，包括处于非活动状态的任务。

- Maximum usable bandwidth (max_bw): This is the maximum bandwidth usable by deadline tasks and is currently set to the RT capacity.

> \*最大可用带宽（max_bw）：这是截止日期任务可用的最大带宽，当前设置为 RT 容量。

The algorithm reclaims the bandwidth of the tasks in Inactive state. It does so by decrementing the runtime of the executing task Ti at a pace equal to

> 该算法回收处于非活动状态的任务的带宽。它通过以等于

dq = -(max{ Ui, (Umax - Uinact - Uextra) } / Umax) dt

where:

- Ui is the bandwidth of task Ti;

- Umax is the maximum reclaimable utilization (subjected to RT throttling limits);

> \*Umax 是最大可回收利用率（受 RT 节流限制）；

- Uinact is the (per runqueue) inactive utilization, computed as (this_bq - running_bw);

> \*Uinact 是（每个运行队列）非活动利用率，计算为（this_bq-running_bw）；

- Uextra is the (per runqueue) extra reclaimable utilization (subjected to RT throttling limits).

> \*Uextra 是（每个运行队列）额外的可回收利用率（受 RT 限制）。

Let's now see a trivial example of two deadline tasks with runtime equal to 4 and period equal to 8 (i.e., bandwidth equal to 0.5):

> 现在，让我们看看两个最后期限任务的一个微不足道的例子，它们的运行时间等于 4，周期等于 8（即带宽等于 0.5）：

```
       A            Task T1
       |
       |                               |
       |                               |
       |--------                       |----
       |       |                       V
       |---|---|---|---|---|---|---|---|--------->t
       0   1   2   3   4   5   6   7   8


       A            Task T2
       |
       |                               |
       |                               |
       |       ------------------------|
       |       |                       V
       |---|---|---|---|---|---|---|---|--------->t
       0   1   2   3   4   5   6   7   8


       A            running_bw
       |
     1 -----------------               ------
       |               |               |
    0.5-               -----------------
       |                               |
       |---|---|---|---|---|---|---|---|--------->t
       0   1   2   3   4   5   6   7   8


- Time t = 0:

  Both tasks are ready for execution and therefore in ActiveContending state.
  Suppose Task T1 is the first task to start execution.
  Since there are no inactive tasks, its runtime is decreased as dq = -1 dt.

- Time t = 2:

  Suppose that task T1 blocks
  Task T1 therefore enters the ActiveNonContending state. Since its remaining
  runtime is equal to 2, its 0-lag time is equal to t = 4.
  Task T2 start execution, with runtime still decreased as dq = -1 dt since
  there are no inactive tasks.

- Time t = 4:

  This is the 0-lag time for Task T1. Since it didn't woken up in the
  meantime, it enters the Inactive state. Its bandwidth is removed from
  running_bw.
  Task T2 continues its execution. However, its runtime is now decreased as
  dq = - 0.5 dt because Uinact = 0.5.
  Task T2 therefore reclaims the bandwidth unused by Task T1.

- Time t = 8:

  Task T1 wakes up. It enters the ActiveContending state again, and the
  running_bw is incremented.


```

### 2.3 Energy-aware scheduling[¶](#energy-aware-scheduling "Permalink to this heading")

When cpufreq's schedutil governor is selected, SCHED_DEADLINE implements the GRUB-PA [19] algorithm, reducing the CPU operating frequency to the minimum value that still allows to meet the deadlines. This behavior is currently implemented only for ARM architectures.

> 当选择 cpufreq 的 schedutil 调控器时，SCHED_DEADLINE 实现 GRUB-PA[19]算法，将 CPU 工作频率降低到最小值，仍然可以满足截止日期。此行为目前仅针对 ARM 体系结构实现。

A particular care must be taken in case the time needed for changing frequency is of the same order of magnitude of the reservation period. In such cases, setting a fixed CPU frequency results in a lower amount of deadline misses.

> 如果改变频率所需的时间与保留期的数量级相同，则必须特别小心。在这种情况下，设置固定的 CPU 频率会导致较少的截止日期未命中。

## 3. Scheduling Real-Time Tasks[¶](#scheduling-real-time-tasks "Permalink to this heading")

Warning

This section contains a (not-thorough) summary on classical deadline scheduling theory, and how it applies to SCHED_DEADLINE. The reader can "safely" skip to Section 4 if only interested in seeing how the scheduling policy can be used. Anyway, we strongly recommend to come back here and continue reading (once the urge for testing is satisfied :P) to be sure of fully understanding all technical details.

> 本节包含对经典截止日期调度理论的（不全面）总结，以及它如何应用于 SCHED_deadline。如果读者只想了解如何使用调度策略，则可以“安全”地跳到第 4 节。无论如何，我们强烈建议回到这里继续阅读（一旦测试的欲望得到满足：P），以确保完全理解所有技术细节。

There are no limitations on what kind of task can exploit this new scheduling discipline, even if it must be said that it is particularly suited for periodic or sporadic real-time tasks that need guarantees on their timing behavior, e.g., multimedia, streaming, control applications, etc.

> 哪种任务可以利用这种新的调度规则没有限制，即使必须说它特别适合于需要保证其时序行为的周期性或偶发性实时任务，例如多媒体、流媒体、控制应用程序等。

### 3.1 Definitions[¶](#definitions "Permalink to this heading")

A typical real-time task is composed of a repetition of computation phases (task instances, or jobs) which are activated on a periodic or sporadic fashion. Each job J*j (where J_j is the j^th job of the task) is characterized by an arrival time r_j (the time when the job starts), an amount of computation time c_j needed to finish the job, and a job absolute deadline d_j, which is the time within which the job should be finished. The maximum execution time max{c_j} is called "Worst Case Execution Time" (WCET) for the task. A real-time task can be periodic with period P if r*{j+1} = r*j + P, or sporadic with minimum inter-arrival time P is r*{j+1} >= r_j + P. Finally, d_j = r_j + D, where D is the task's relative deadline. Summing up, a real-time task can be described as

> 典型的实时任务由重复的计算阶段（任务实例或作业）组成，这些阶段以周期性或偶发的方式激活。每个作业 J*J（其中 J_J 是任务的第 J 个作业）由到达时间 r_J（作业开始的时间）、完成作业所需的计算时间 c_J 和作业绝对截止日期 d_J 表征，作业绝对截止时间 d_J 是作业应该完成的时间。最大执行时间 max｛c_j｝被称为任务的“最坏情况执行时间”（WCET）。如果 r*{j+1}＝ r*j+P，则实时任务可以是周期性的，周期为 P，或者具有最小到达时间 P 的偶发任务是 r*{j+1}＞＝ r_j+P。最后，d_j=r_j+d，其中 d 是任务的相对截止日期。总之，实时任务可以描述为

Task = (WCET, D, P)

The utilization of a real-time task is defined as the ratio between its WCET and its period (or minimum inter-arrival time), and represents the fraction of CPU time needed to execute the task.

> 实时任务的利用率被定义为其 WCET 与其周期（或最小到达时间）之间的比率，并表示执行任务所需的 CPU 时间的分数。

If the total utilization U=sum(WCET_i/P_i) is larger than M (with M equal to the number of CPUs), then the scheduler is unable to respect all the deadlines. Note that total utilization is defined as the sum of the utilizations WCET_i/P_i over all the real-time tasks in the system. When considering multiple real-time tasks, the parameters of the i-th task are indicated with the "\_i" suffix. Moreover, if the total utilization is larger than M, then we risk starving non- real-time tasks by real-time tasks. If, instead, the total utilization is smaller than M, then non real-time tasks will not be starved and the system might be able to respect all the deadlines. As a matter of fact, in this case it is possible to provide an upper bound for tardiness (defined as the maximum between 0 and the difference between the finishing time of a job and its absolute deadline). More precisely, it can be proven that using a global EDF scheduler the maximum tardiness of each task is smaller or equal than

> 如果总利用率 U ＝ sum（WCET_i/P_i）大于 M（其中 M 等于 CPU 的数量），则调度器不能遵守所有截止日期。注意，总利用率被定义为系统中所有实时任务的利用率 WCET_i/P_i 的总和。当考虑多个实时任务时，第 i 个任务的参数用“\_i”后缀表示。此外，如果总利用率大于 M，那么我们就有实时任务饿死非实时任务的风险。相反，如果总利用率小于 M，那么非实时任务将不会匮乏，系统可能能够遵守所有截止日期。事实上，在这种情况下，可以提供延迟的上限（定义为 0 和作业完成时间与其绝对截止日期之差之间的最大值）。更准确地说，可以证明使用全局 EDF 调度器，每个任务的最大延迟小于或等于

((M − 1) · WCET_max − WCET_min)/(M − (M − 2) · U_max) + WCET_max

where WCET_max = max{WCET_i} is the maximum WCET, WCET_min=min{WCET_i} is the minimum WCET, and U_max = max{WCET_i/P_i} is the maximum utilization[12].

> 其中 WCET_max=max{WCET_i}是最大 WCET，WCET_min=min{WCET_id}是最小 WCET，U_max=max{WCET \_i/P_i}是最高利用率[12]。

### 3.2 Schedulability Analysis for Uniprocessor Systems[¶](#schedulability-analysis-for-uniprocessor-systems "Permalink to this heading")

If M=1 (uniprocessor system), or in case of partitioned scheduling (each real-time task is statically assigned to one and only one CPU), it is possible to formally check if all the deadlines are respected. If D_i = P_i for all tasks, then EDF is able to respect all the deadlines of all the tasks executing on a CPU if and only if the total utilization of the tasks running on such a CPU is smaller or equal than 1. If D_i != P_i for some task, then it is possible to define the density of a task as WCET_i/min{D_i,P_i}, and EDF is able to respect all the deadlines of all the tasks running on a CPU if the sum of the densities of the tasks running on such a CPU is smaller or equal than 1:

> 如果 M ＝ 1（单处理器系统），或者在分区调度的情况下（每个实时任务静态分配给一个且只有一个 CPU），可以正式检查是否遵守了所有截止日期。如果对于所有任务 D_i ＝ P_i，则 EDF 能够遵守在 CPU 上执行的所有任务的所有截止日期，当且仅当在这样的 CPU 上运行的任务的总利用率小于或等于 1。如果 D_i！=对于某个任务 P_i，则可以将任务的密度定义为 WCET_i/min{D_i，P_i}，并且如果在 CPU 上运行的任务的密度之和小于或等于 1:

sum(WCET_i / min{D_i, P_i}) <= 1

It is important to notice that this condition is only sufficient, and not necessary: there are task sets that are schedulable, but do not respect the condition. For example, consider the task set {Task_1,Task_2} composed by Task_1=(50ms,50ms,100ms) and Task_2=(10ms,100ms,100ms). EDF is clearly able to schedule the two tasks without missing any deadline (Task_1 is scheduled as soon as it is released, and finishes just in time to respect its deadline; Task_2 is scheduled immediately after Task_1, hence its response time cannot be larger than 50ms + 10ms = 60ms) even if

> 需要注意的是，这个条件只是充分的，而不是必要的：有些任务集是可调度的，但不尊重这个条件。例如，考虑由 task_1=（50ms，50ms，100ms）和 task_2=（10ms，100ms.）组成的任务集{task_1，task_2}。EDF 显然能够在不错过任何截止日期的情况下安排这两项任务（Task_1 在发布后立即安排，并在截止日期前及时完成；Task_2 在 Task_1 之后立即安排，因此其响应时间不能大于 50ms+10ms=60ms），即使

50 / min{50,100} + 10 / min{100, 100} = 50 / 50 + 10 / 100 = 1.1

Of course it is possible to test the exact schedulability of tasks with D_i != P_i (checking a condition that is both sufficient and necessary), but this cannot be done by comparing the total utilization or density with a constant. Instead, the so called "processor demand" approach can be used, computing the total amount of CPU time h(t) needed by all the tasks to respect all of their deadlines in a time interval of size t, and comparing such a time with the interval size t. If h(t) is smaller than t (that is, the amount of time needed by the tasks in a time interval of size t is smaller than the size of the interval) for all the possible values of t, then EDF is able to schedule the tasks respecting all of their deadlines. Since performing this check for all possible values of t is impossible, it has been proven[4,5,6] that it is sufficient to perform the test for values of t between 0 and a maximum value L. The cited papers contain all of the mathematical details and explain how to compute h(t) and L. In any case, this kind of analysis is too complex as well as too time-consuming to be performed on-line. Hence, as explained in Section 4 Linux uses an admission test based on the tasks' utilizations.

> 当然，使用 D_i！=可以测试任务的精确可调度性 P_i（检查一个既充分又必要的条件），但这不能通过将总利用率或密度与常数进行比较来实现。相反，可以使用所谓的“处理器需求”方法，计算所有任务在大小为 t 的时间间隔内遵守其所有截止日期所需的 CPU 时间总量 h（t），并将该时间与间隔大小 t 进行比较。如果对于 t 的所有可能值，h（t）小于 t（即，在大小为 t 的时间间隔中任务所需的时间量小于间隔的大小），则 EDF 能够按照任务的所有截止日期来调度任务。由于不可能对 t 的所有可能值进行这种检查，已经证明[4，5，6]对 0 和最大值 L 之间的 t 值进行测试就足够了。引用的论文包含了所有的数学细节，并解释了如何计算 h（t）和 L。无论如何，这种分析太复杂，也太耗时，无法在线进行。因此，如第 4 节所述，Linux 使用基于任务利用率的准入测试。

### 3.3 Schedulability Analysis for Multiprocessor Systems[¶](#schedulability-analysis-for-multiprocessor-systems "Permalink to this heading")

On multiprocessor systems with global EDF scheduling (non partitioned systems), a sufficient test for schedulability can not be based on the utilizations or densities: it can be shown that even if D_i = P_i task sets with utilizations slightly larger than 1 can miss deadlines regardless of the number of CPUs.

> 在具有全局 EDF 调度的多处理器系统（非分区系统）上，对可调度性的充分测试不能基于利用率或密度：可以表明，即使利用率略大于 1 的 D_i=P_i 任务集也可以错过截止日期，而与 CPU 数量无关。

Consider a set {Task*1,...Task*{M+1}} of M+1 tasks on a system with M CPUs, with the first task Task*1=(P,P,P) having period, relative deadline and WCET equal to P. The remaining M tasks Task_i=(e,P-1,P-1) have an arbitrarily small worst case execution time (indicated as "e" here) and a period smaller than the one of the first task. Hence, if all the tasks activate at the same time t, global EDF schedules these M tasks first (because their absolute deadlines are equal to t + P - 1, hence they are smaller than the absolute deadline of Task_1, which is t + P). As a result, Task_1 can be scheduled only at time t + e, and will finish at time t + e + P, after its absolute deadline. The total utilization of the task set is U = M · e / (P - 1) + P / P = M · e / (P - 1) + 1, and for small values of e this can become very close to 1. This is known as "Dhall's effect"[7]. Note: the example in the original paper by Dhall has been slightly simplified here (for example, Dhall more correctly computed lim*{e->0}U).

> 考虑在具有 M 个 CPU 的系统上 M+1 个任务的一组{Task*1，…Task*{M+1}}，其中第一个任务 Task*1=（P，P）具有周期、相对截止日期和等于 P 的 WCET。其余 M 个任务 Task_i=（e，P-1，P-1）具有任意小的最坏情况执行时间（此处表示为“e”）和小于第一个任务的周期。因此，如果所有任务都在同一时间 t 激活，全局 EDF 首先调度这 M 个任务（因为它们的绝对截止日期等于 t+P-1，因此它们小于 Task_1 的绝对截止时间，即 t+P）。因此，Task_1 只能在时间 t+e 进行调度，并且将在其绝对截止日期之后的时间 t+e+P 完成。任务集的总利用率为 U=M·e/（P-1）+P=M·e/（P-1）+1，对于 e 的小值，这可能变得非常接近 1。这被称为“达尔效应”[7]。注意：Dhall 在原始论文中的例子在这里被稍微简化了（例如，Dhall 更正确地计算了 lim*{e->0}U）。

More complex schedulability tests for global EDF have been developed in real-time literature[8,9], but they are not based on a simple comparison between total utilization (or density) and a fixed constant. If all tasks have D_i = P_i, a sufficient schedulability condition can be expressed in a simple way:

> 实时文献[8，9]中已经开发了更复杂的全局 EDF 可调度性测试，但它们并不是基于总利用率（或密度）和固定常数之间的简单比较。如果所有任务都有 D_i=P_i，则一个充分的可调度性条件可以用一种简单的方式表示：

sum(WCET_i / P_i) <= M - (M - 1) · U_max

where U_max = max{WCET_i / P_i}[10]. Notice that for U_max = 1, M - (M - 1) · U_max becomes M - M + 1 = 1 and this schedulability condition just confirms the Dhall's effect. A more complete survey of the literature about schedulability tests for multi-processor real-time scheduling can be found in [11].

> 其中 U\_ max ＝ max{WCET_i/P_i}[10]。注意，对于 U_max=1，M-（M-1）·U_max 变为 M-M+1=1，这个可调度性条件正好证实了 Dhall 的效应。关于多处理器实时调度的可调度性测试的文献，可以在[11]中找到更完整的调查。

As seen, enforcing that the total utilization is smaller than M does not guarantee that global EDF schedules the tasks without missing any deadline (in other words, global EDF is not an optimal scheduling algorithm). However, a total utilization smaller than M is enough to guarantee that non real-time tasks are not starved and that the tardiness of real-time tasks has an upper bound[12] (as previously noted). Different bounds on the maximum tardiness experienced by real-time tasks have been developed in various papers[13,14], but the theoretical result that is important for SCHED_DEADLINE is that if the total utilization is smaller or equal than M then the response times of the tasks are limited.

> 如图所示，强制总利用率小于 M 并不能保证全局 EDF 在不错过任何截止日期的情况下调度任务（换句话说，全局 EDF 不是最优调度算法）。然而，小于 M 的总利用率足以保证非实时任务不会匮乏，并且实时任务的延迟性有上限[12]（如前所述）。各种论文[13，14]对实时任务所经历的最大延迟提出了不同的界限，但对 SCHED_DEADLINE 来说重要的理论结果是，如果总利用率小于或等于 M，则任务的响应时间是有限的。

### 3.4 Relationship with SCHED_DEADLINE Parameters[¶](#relationship-with-sched-deadline-parameters "Permalink to this heading")

Finally, it is important to understand the relationship between the SCHED_DEADLINE scheduling parameters described in Section 2 (runtime, deadline and period) and the real-time task parameters (WCET, D, P) described in this section. Note that the tasks' temporal constraints are represented by its absolute deadlines d_j = r_j + D described above, while SCHED_DEADLINE schedules the tasks according to scheduling deadlines (see Section 2). If an admission test is used to guarantee that the scheduling deadlines are respected, then SCHED_DEADLINE can be used to schedule real-time tasks guaranteeing that all the jobs' deadlines of a task are respected. In order to do this, a task must be scheduled by setting:

> 最后，重要的是要理解第 2 节中描述的 SCHED_DEADLINE 调度参数（运行时间、截止日期和周期）与本节中描述了的实时任务参数（WCET、D、P）之间的关系。请注意，任务的时间约束由其绝对截止日期 d_j=r_j+d 表示，如上所述，而 SCHED_DEADLINE 根据调度截止日期来调度任务（见第 2 节）。如果使用准入测试来确保遵守调度截止日期，则可以使用 SCHED_DEADLINE 来调度实时任务，以确保遵守任务的所有作业的截止日期。要执行此操作，必须通过设置来安排任务：

- runtime >= WCET
- deadline = D
- period <= P

IOW, if runtime >= WCET and if period is <= P, then the scheduling deadlines and the absolute deadlines (d_j) coincide, so a proper admission control allows to respect the jobs' absolute deadlines for this task (this is what is called "hard schedulability property" and is an extension of Lemma 1 of [2]). Notice that if runtime > deadline the admission control will surely reject this task, as it is not possible to respect its temporal constraints.

> IOW，如果运行时>=WCET，并且周期<=P，则调度截止日期和绝对截止日期（d_j）重合，因此适当的准入控制允许尊重作业对该任务的绝对截止日期。请注意，如果运行时>截止日期，准入控制肯定会拒绝此任务，因为它不可能遵守其时间限制。

References:

1 - C. L. Liu and J. W. Layland. Scheduling algorithms for multiprogram-

ming in a hard-real-time environment. Journal of the Association for Computing Machinery, 20(1), 1973.

> 明在一个艰难的实时环境中。《计算机协会杂志》，20（1），1973 年。

2 - L. Abeni , G. Buttazzo. Integrating Multimedia Applications in Hard

Real-Time Systems. Proceedings of the 19th IEEE Real-time Systems Symposium, 1998. [http://retis.sssup.it/~giorgio/paps/1998/rtss98-cbs.pdf](http://retis.sssup.it/~giorgio/paps/1998/rtss98-cbs.pdf)

> 实时系统。第 19 届 IEEE 实时系统研讨会论文集，1998 年。[http://retis.sssup.it/~giorgio/paps/1998/rtss98 cbs.pdf](http://retis.sssup.it/~giorgio/paps/1998/rtss98 cbs.pdf）

3 - L. Abeni. Server Mechanisms for Multimedia Applications. ReTiS Lab

Technical Report. [http://disi.unitn.it/~abeni/tr-98-01.pdf](http://disi.unitn.it/~abeni/tr-98-01.pdf)

> 技术报告。[http://disi.unitn.it/~abeni/tr-98-01.pdf(http://disi.unitn.it/~abeni/tr-98-01.pdf）

4 - J. Y. Leung and M.L. Merril. A Note on Preemptive Scheduling of

Periodic, Real-Time Tasks. Information Processing Letters, vol. 11, no. 3, pp. 115-118, 1980.

> 定期实时任务。《信息处理函件》，第 11 卷，第 3 期，第 115-1181980 页。

5 - S. K. Baruah, A. K. Mok and L. E. Rosier. Preemptively Scheduling

Hard-Real-Time Sporadic Tasks on One Processor. Proceedings of the 11th IEEE Real-time Systems Symposium, 1990.

> 一个处理器上的硬实时零星任务。第 11 届 IEEE 实时系统研讨会论文集，1990 年。

6 - S. K. Baruah, L. E. Rosier and R. R. Howell. Algorithms and Complexity

> 6-S.K.Baruah、L.E.Rosier 和 R.R.Howell。算法与复杂性

Concerning the Preemptive Scheduling of Periodic Real-Time tasks on One Processor. Real-Time Systems Journal, vol. 4, no. 2, pp 301-324, 1990.

> 关于在一个处理器上周期性实时任务的抢占调度。《实时系统杂志》，第 4 卷，第 2 期，第 301-3241990 页。

7 - S. J. Dhall and C. L. Liu. On a real-time scheduling problem. Operations

> 7-S.J.Dhall 和 C.L.Liu。关于实时调度问题。操作

research, vol. 26, no. 1, pp 127-140, 1978.

8 - T. Baker. Multiprocessor EDF and Deadline Monotonic Schedulability

Analysis. Proceedings of the 24th IEEE Real-Time Systems Symposium, 2003.

9 - T. Baker. An Analysis of EDF Schedulability on a Multiprocessor.

IEEE Transactions on Parallel and Distributed Systems, vol. 16, no. 8, pp 760-768, 2005.

> 《IEEE 并行和分布式系统汇刊》，第 16 卷，第 8 期，第 760-7682005 页。

10 - J. Goossens, S. Funk and S. Baruah, Priority-Driven Scheduling of

Periodic Task Systems on Multiprocessors. Real-Time Systems Journal, vol. 25, no. 2–3, pp. 187–205, 2003.

> 多处理器上的周期任务系统。《实时系统杂志》，第 25 卷，第 2–3 期，第 187–205 页，2003 年。

11 - R. Davis and A. Burns. A Survey of Hard Real-Time Scheduling for

Multiprocessor Systems. ACM Computing Surveys, vol. 43, no. 4, 2011. [http://www-users.cs.york.ac.uk/~robdavis/papers/MPSurveyv5.0.pdf](http://www-users.cs.york.ac.uk/~robdavis/papers/MPSurveyv5.0.pdf)

> 多处理器系统。ACM 计算调查，第 43 卷，第 4 期，2011 年。[http://www-users.cs.york.ac.uk/~robdavis/papers/MPSurveyv5.0.pdf](http://www-users.cs.york.ac.uk/~robdavis/papers/MPSurveyv5.0.pdf）

12 - U. C. Devi and J. H. Anderson. Tardiness Bounds under Global EDF

Scheduling on a Multiprocessor. Real-Time Systems Journal, vol. 32, no. 2, pp 133-189, 2008.

> 在多处理器上进行调度。《实时系统杂志》，第 32 卷，第 2 期，第 133-1892008 页。

13 - P. Valente and G. Lipari. An Upper Bound to the Lateness of Soft

Real-Time Tasks Scheduled by EDF on Multiprocessors. Proceedings of the 26th IEEE Real-Time Systems Symposium, 2005.

> EDF 在多处理器上调度的实时任务。第 26 届 IEEE 实时系统研讨会论文集，2005 年。

14 - J. Erickson, U. Devi and S. Baruah. Improved tardiness bounds for

Global EDF. Proceedings of the 22nd Euromicro Conference on Real-Time Systems, 2010.

> 全球 EDF。第 22 届实时系统欧洲微会议论文集，2010 年。

15 - G. Lipari, S. Baruah, Greedy reclamation of unused bandwidth in

constant-bandwidth servers, 12th IEEE Euromicro Conference on Real-Time Systems, 2000.

> 恒定带宽服务器，第 12 届 IEEE 实时系统欧洲微会议，2000 年。

16 - L. Abeni, J. Lelli, C. Scordino, L. Palopoli, Greedy CPU reclaiming for

> 16-L.Abeni，J.Lelli，C.Scordino，L.Palopoli，贪婪的 CPU 回收

SCHED DEADLINE. In Proceedings of the Real-Time Linux Workshop (RTLWS), Dusseldorf, Germany, 2014.

> SCHED 截止日期。《实时 Linux 研讨会论文集》（RTLWS），德国杜塞尔多夫，2014 年。

17 - L. Abeni, G. Lipari, A. Parri, Y. Sun, Multicore CPU reclaiming: parallel

> 17-L.Abeni，G.Lipari，A.Parri，Y.Sun，多核 CPU 回收：并行

or sequential?. In Proceedings of the 31st Annual ACM Symposium on Applied Computing, 2016.

18 - J. Lelli, C. Scordino, L. Abeni, D. Faggioli, Deadline scheduling in the

> 18-J.Lelli、C.Scordino、L.Abeni、D.Faggioli

Linux kernel, Software: Practice and Experience, 46(6): 821-839, June 2016.

> Linux 内核，软件：实践与经验，46（6）：821-8392016 年 6 月。

19 - C. Scordino, L. Abeni, J. Lelli, Energy-Aware Real-Time Scheduling in

> 19-C.Scordino，L.Abeni，J.Lelli，能源感知实时调度

the Linux Kernel, 33rd ACM/SIGAPP Symposium On Applied Computing (SAC 2018), Pau, France, April 2018.

> Linux 内核，第 33 届 ACM/SIGAPP 应用计算研讨会（SAC 2018），法国波城，2018 年 4 月。

## 4. Bandwidth management[¶](#bandwidth-management "Permalink to this heading")

As previously mentioned, in order for -deadline scheduling to be effective and useful (that is, to be able to provide "runtime" time units within "deadline"), it is important to have some method to keep the allocation of the available fractions of CPU time to the various tasks under control. This is usually called "admission control" and if it is not performed, then no guarantee can be given on the actual scheduling of the -deadline tasks.

> 如前所述，为了使截止日期调度有效和有用（也就是说，能够在“截止日期”内提供“运行时”时间单位），重要的是要有一些方法来控制 CPU 时间的可用部分分配给各种任务。这通常被称为“准入控制”，如果不执行，则无法保证截止日期任务的实际调度。

As already stated in Section 3, a necessary condition to be respected to correctly schedule a set of real-time tasks is that the total utilization is smaller than M. When talking about -deadline tasks, this requires that the sum of the ratio between runtime and period for all tasks is smaller than M. Notice that the ratio runtime/period is equivalent to the utilization of a "traditional" real-time task, and is also often referred to as "bandwidth". The interface used to control the CPU bandwidth that can be allocated to -deadline tasks is similar to the one already used for -rt tasks with real-time group scheduling (a.k.a. RT-throttling - see [Real-Time group scheduling](https://docs.kernel.org/scheduler/sched-rt-group.html)), and is based on readable/ writable control files located in procfs (for system wide settings). Notice that per-group settings (controlled through cgroupfs) are still not defined for -deadline tasks, because more discussion is needed in order to figure out how we want to manage SCHED_DEADLINE bandwidth at the task group level.

> 如第 3 节所述，正确调度一组实时任务需要满足的一个必要条件是总利用率小于 M。当谈到截止日期任务时，这要求所有任务的运行时间和周期之和小于 M。请注意，运行时/周期的比率相当于“传统”实时任务的利用率，也通常被称为“带宽”。用于控制可分配给-dermine 任务的 CPU 带宽的接口与已经用于具有实时组调度的-rt 任务的接口类似（也称为 rt 节流-请参阅[实时组调度](https://docs.kernel.org/scheduler/sched-rt-group.html))，并且基于位于 procfs 中的可读/可写控制文件（用于系统范围的设置）。请注意，每个组的设置（通过 cgroupfs 控制）仍然没有为-dialdline 任务定义，因为需要更多的讨论才能弄清楚我们希望如何在任务组级别管理 SCHED_deadline 带宽。

A main difference between deadline bandwidth management and RT-throttling is that -deadline tasks have bandwidth on their own (while -rt ones don't!), and thus we don't need a higher level throttling mechanism to enforce the desired bandwidth. In other words, this means that interface parameters are only used at admission control time (i.e., when the user calls sched_setattr()). Scheduling is then performed considering actual tasks' parameters, so that CPU bandwidth is allocated to SCHED_DEADLINE tasks respecting their needs in terms of granularity. Therefore, using this simple interface we can put a cap on total utilization of -deadline tasks (i.e., Sum (runtime_i / period_i) < global_dl_utilization_cap).

> 截止日期带宽管理和 RT 节流之间的主要区别在于-截止日期任务有自己的带宽（而-RT 任务没有！），因此我们不需要更高级别的节流机制来强制执行所需的带宽。换句话说，这意味着接口参数只在准入控制时使用（即，当用户调用 sched_settr（）时）。然后，考虑实际任务的参数来执行调度，以便将 CPU 带宽分配给 SCHED_DEADLINE 任务，并考虑到它们在粒度方面的需求。因此，使用这个简单的接口，我们可以限制-dermine 任务的总利用率（即，Sum（runtime_i/paperiod_i）<global_dl_utilization_cap）。

### 4.1 System wide settings[¶](#system-wide-settings "Permalink to this heading")

The system wide settings are configured under the /proc virtual file system.

> 系统范围的设置是在/proc 虚拟文件系统下配置的。

For now the -rt knobs are used for -deadline admission control and the -deadline runtime is accounted against the -rt runtime. We realize that this isn't entirely desirable; however, it is better to have a small interface for now, and be able to change it easily later. The ideal situation (see 5.) is to run -rt tasks from a -deadline server; in which case the -rt bandwidth is a direct subset of dl_bw.

> 目前，-rt 旋钮用于-dateline 准入控制，-dateline 运行时与-rt 运行时相对应。我们意识到这并不完全令人满意；不过，现在最好有一个小接口，以后可以轻松更改。理想的情况（请参见 5.）是从-dermine 服务器运行-rt 任务；在这种情况下，-rt 带宽是 dlbw 的直接子集。

This means that, for a root_domain comprising M CPUs, -deadline tasks can be created while the sum of their bandwidths stays below:

> 这意味着，对于包含 M 个 CPU 的 root_domain，可以在带宽总和保持在以下的情况下创建-dermine 任务：

M \* (sched_rt_runtime_us / sched_rt_period_us)

It is also possible to disable this bandwidth management logic, and be thus free of oversubscribing the system up to any arbitrary level. This is done by writing -1 in /proc/sys/kernel/sched_rt_runtime_us.

> 还可以禁用该带宽管理逻辑，从而避免将系统过度订阅到任何任意级别。这是通过在/proc/sys/kernel/sched_rt_runtime_us 中写入-1 来完成的。

### 4.2 Task interface[¶](#task-interface "Permalink to this heading")

Specifying a periodic/sporadic task that executes for a given amount of runtime at each instance, and that is scheduled according to the urgency of its own timing constraints needs, in general, a way of declaring:

> 指定一个周期性/偶发性任务，该任务在每个实例上执行给定的运行时间，并根据其自身时间限制的紧迫性进行调度，通常需要一种声明方式：

- a (maximum/typical) instance execution time,
- a minimum interval between consecutive instances,
- a time constraint by which each instance must be completed.

Therefore:

- a new struct sched_attr, containing all the necessary fields is provided;

> \*提供了一个新的结构体 sched_attr，包含所有必要的字段；

- the new scheduling related syscalls that manipulate it, i.e., sched_setattr() and sched_getattr() are implemented.

> \*实现了操作它的新的与调度相关的系统调用，即 sched_settr（）和 schedgetattr（）。

For debugging purposes, the leftover runtime and absolute deadline of a SCHED_DEADLINE task can be retrieved through /proc/<pid>/sched (entries dl.runtime and dl.deadline, both values in ns). A programmatic way to retrieve these values from production code is under discussion.

> 出于调试目的，SCHED_deadline 任务的剩余运行时和绝对截止日期可以通过/proc/<pid>/SCHED（条目 dl.runtime 和 dl.deadline，均以 ns 为单位）检索。正在讨论从生产代码中检索这些值的编程方法。

### 4.3 Default behavior[¶](#default-behavior "Permalink to this heading")

The default value for SCHED_DEADLINE bandwidth is to have rt_runtime equal to 950000. With rt_period equal to 1000000, by default, it means that -deadline tasks can use at most 95%, multiplied by the number of CPUs that compose the root_domain, for each root_domain. This means that non -deadline tasks will receive at least 5% of the CPU time, and that -deadline tasks will receive their runtime with a guaranteed worst-case delay respect to the "deadline" parameter. If "deadline" = "period" and the cpuset mechanism is used to implement partitioned scheduling (see Section 5), then this simple setting of the bandwidth management is able to deterministically guarantee that -deadline tasks will receive their runtime in a period.

> SCHED_DEADLINE 带宽的默认值是 rt_runtime 等于 950000。默认情况下，rt_period 等于 1000000，这意味着-dermine 任务最多可以使用 95%，乘以组成 root_domain 的 CPU 数量，用于每个 root_domain。这意味着非截止日期任务将获得至少 5%的 CPU 时间，而截止日期任务的运行时间将保证在“截止日期”参数的最坏情况下延迟。如果“deadline”=“period”，并且使用 cpuset 机制来实现分区调度（请参阅第 5 节），那么这种简单的带宽管理设置能够决定性地保证-dateline 任务将在一段时间内接收其运行时。

Finally, notice that in order not to jeopardize the admission control a -deadline task cannot fork.

> 最后，请注意，为了不危及录取控制，截止日期任务不能分叉。

### 4.4 Behavior of sched_yield()[¶](#behavior-of-sched-yield "Permalink to this heading")

When a SCHED_DEADLINE task calls sched_yield(), it gives up its remaining runtime and is immediately throttled, until the next period, when its runtime will be replenished (a special flag dl_yielded is set and used to handle correctly throttling and runtime replenishment after a call to sched_yield()).

> 当 SCHED_DEADLINE 任务调用 SCHED_yield（）时，它会放弃剩余的运行时，并立即被限制，直到下一个周期，它的运行时才会被补充（设置了一个特殊的标志 dl_yield，用于在调用 SCHED_yield（）后正确处理限制和运行时补充）。

This behavior of sched_yield() allows the task to wake-up exactly at the beginning of the next period. Also, this may be useful in the future with bandwidth reclaiming mechanisms, where sched_yield() will make the leftoever runtime available for reclamation by other SCHED_DEADLINE tasks.

> sched_yield（）的这种行为允许任务在下一个周期开始时准确地唤醒。此外，这在未来的带宽回收机制中可能很有用，其中 sched_yield（）将使 leftevery 运行时可用于其他 sched_DEADLINE 任务的回收。

## 5. Tasks CPU affinity[¶](#tasks-cpu-affinity "Permalink to this heading")

-deadline tasks cannot have an affinity mask smaller that the entire root_domain they are created on. However, affinities can be specified through the cpuset facility ([CPUSETS](https://docs.kernel.org/admin-guide/cgroup-v1/cpusets.html)).

> -截止日期任务的关联掩码不能小于创建任务的整个 root_domain。但是，可以通过 cpuset 工具（[CPUSETS]）指定关联(https://docs.kernel.org/admin-guide/cgroup-v1/cpusets.html))。

### 5.1 SCHED_DEADLINE and cpusets HOWTO[¶](#sched-deadline-and-cpusets-howto "Permalink to this heading")

An example of a simple configuration (pin a -deadline task to CPU0) follows (rt-app is used to create a -deadline task):

> 下面是一个简单配置的示例（将-dermine 任务固定到 CPU0）（rt 应用程序用于创建-dermineline 任务）：

```
mkdir /dev/cpuset
mount -t cgroup -o cpuset cpuset /dev/cpuset
cd /dev/cpuset
mkdir cpu0
echo 0 > cpu0/cpuset.cpus
echo 0 > cpu0/cpuset.mems
echo 1 > cpuset.cpu_exclusive
echo 0 > cpuset.sched_load_balance
echo 1 > cpu0/cpuset.cpu_exclusive
echo 1 > cpu0/cpuset.mem_exclusive
echo $$ > cpu0/tasks
rt-app -t 100000:10000:d:0 -D5 # it is now actually superfluous to specify
                               # task affinity


```

## 6. Future plans[¶](#future-plans "Permalink to this heading")

Still missing:

- programmatic way to retrieve current runtime and absolute deadline

- refinements to deadline inheritance, especially regarding the possibility of retaining bandwidth isolation among non-interacting tasks. This is being studied from both theoretical and practical points of view, and hopefully we should be able to produce some demonstrative code soon;

> \*对截止日期继承的改进，特别是关于在非交互任务之间保持带宽隔离的可能性。这是从理论和实践的角度进行研究的，希望我们能够很快产生一些演示代码；

- (c)group based bandwidth management, and maybe scheduling;

- access control for non-root users (and related security concerns to address), which is the best way to allow unprivileged use of the mechanisms and how to prevent non-root users "cheat" the system?

> \*非 root 用户的访问控制（以及需要解决的相关安全问题），哪种方式是允许非特权使用该机制的最佳方式，以及如何防止非 root 用户“欺骗”系统？

As already discussed, we are planning also to merge this work with the EDF throttling patches [[https://lore.kernel.org/r/cover.1266931410.git.fabio@helm.retis](https://lore.kernel.org/r/cover.1266931410.git.fabio@helm.retis)] but we still are in the preliminary phases of the merge and we really seek feedback that would help us decide on the direction it should take.

> 如前所述，我们还计划将这项工作与 EDF 节流补丁合并[[https://lore.kernel.org/r/cover.1266931410.git.fabio@helm.retis](https://lore.kernel.org/r/cover.1266931410.git.fabio@helm.retis）]，但我们仍处于合并的初步阶段，我们确实在寻求反馈，以帮助我们决定合并的方向。

## Appendix A. Test suite[¶](#appendix-a-test-suite "Permalink to this heading")

The SCHED_DEADLINE policy can be easily tested using two applications that are part of a wider Linux Scheduler validation suite. The suite is available as a GitHub repository: [https://github.com/scheduler-tools](https://github.com/scheduler-tools).

> SCHED_DEADLINE 策略可以使用两个应用程序轻松地进行测试，这两个应用是更广泛的 Linux Scheduler 验证套件的一部分。该套件可作为 GitHub 存储库使用：[https://github.com/scheduler-tools](https://github.com/scheduler-tools)。

The first testing application is called rt-app and can be used to start multiple threads with specific parameters. rt-app supports SCHED\_{OTHER,FIFO,RR,DEADLINE} scheduling policies and their related parameters (e.g., niceness, priority, runtime/deadline/period). rt-app is a valuable tool, as it can be used to synthetically recreate certain workloads (maybe mimicking real use-cases) and evaluate how the scheduler behaves under such workloads. In this way, results are easily reproducible. rt-app is available at: [https://github.com/scheduler-tools/rt-app](https://github.com/scheduler-tools/rt-app).

> 第一个测试应用程序称为 rt-app，可用于启动具有特定参数的多个线程。rt 应用程序支持 SCHED\_{OTHER、FIFO、RR、DEADLINE}调度策略及其相关参数（例如，精细度、优先级、运行时间/截止日期/周期）。rt-app 是一个很有价值的工具，因为它可以用来综合重新创建某些工作负载（可能模仿真实的用例），并评估调度器在这些工作负载下的行为。通过这种方式，结果很容易重复。rt 应用程序位于：[https://github.com/scheduler-tools/rt-app](https://github.com/scheduler-tools/rt-app)。

Thread parameters can be specified from the command line, with something like this:

> 线程参数可以从命令行指定，如下所示：

```
# rt-app -t 100000:10000:d -t 150000:20000:f:10 -D5


```

The above creates 2 threads. The first one, scheduled by SCHED_DEADLINE, executes for 10ms every 100ms. The second one, scheduled at SCHED_FIFO priority 10, executes for 20ms every 150ms. The test will run for a total of 5 seconds.

> 上面创建了 2 个线程。第一个由 SCHED_DEADLINE 调度，每 100ms 执行 10ms。第二个以 SCHED_FIFO 优先级 10 调度，每 150ms 执行 20ms。测试将总共运行 5 秒钟。

More interestingly, configurations can be described with a json file that can be passed as input to rt-app with something like this:

> 更有趣的是，配置可以用 json 文件来描述，该文件可以作为输入传递给 rt 应用程序，如下所示：

The parameters that can be specified with the second method are a superset of the command line options. Please refer to rt-app documentation for more details (<rt-app-sources>/doc/\*.json).

> 可以使用第二种方法指定的参数是命令行选项的超集。有关更多详细信息，请参阅 rt 应用程序文档（<rt 应用程序源>/doc/\*.json）。

The second testing application is a modification of schedtool, called schedtool-dl, which can be used to setup SCHED_DEADLINE parameters for a certain pid/application. schedtool-dl is available at: [https://github.com/scheduler-tools/schedtool-dl.git](https://github.com/scheduler-tools/schedtool-dl.git).

> 第二个测试应用程序是对 schedtool 的修改，称为 schedtool dl，可用于为某个 pid/应用程序设置 SCHED_DEADLINE 参数。schedtool dl 位于：[https://github.com/scheduler-tools/schedtool-dl.git](https://github.com/scheduler-tools/schedtool-dl.git)。

The usage is straightforward:

```
# schedtool -E -t 10000000:100000000 -e ./my_cpuhog_app


```

With this, my_cpuhog_app is put to run inside a SCHED_DEADLINE reservation of 10ms every 100ms (note that parameters are expressed in microseconds). You can also use schedtool to create a reservation for an already running application, given that you know its pid:

> 这样，my_cpuhog_app 将在每 100ms 10ms 的 SCHED_DEADLINE 保留内运行（注意，参数以微秒表示）。您还可以使用 schedtool 为已经运行的应用程序创建保留，前提是您知道它的 pid：

```
# schedtool -E -t 10000000:100000000 my_app_pid


```

## Appendix B. Minimal main()[¶](#appendix-b-minimal-main "Permalink to this heading")

We provide in what follows a simple (ugly) self-contained code snippet showing how SCHED_DEADLINE reservations can be created by a real-time application developer:

> 我们在下面提供了一个简单（丑陋）的自包含代码片段，展示了实时应用程序开发人员如何创建 SCHED_DEADLINE 保留：

```
#define _GNU_SOURCE
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <linux/unistd.h>
#include <linux/kernel.h>
#include <linux/types.h>
#include <sys/syscall.h>
#include <pthread.h>

#define gettid() syscall(__NR_gettid)

#define SCHED_DEADLINE       6

/* XXX use the proper syscall numbers */
#ifdef __x86_64__
#define __NR_sched_setattr           314
#define __NR_sched_getattr           315
#endif

#ifdef __i386__
#define __NR_sched_setattr           351
#define __NR_sched_getattr           352
#endif

#ifdef __arm__
#define __NR_sched_setattr           380
#define __NR_sched_getattr           381
#endif

static volatile int done;

struct sched_attr {
     __u32 size;

     __u32 sched_policy;
     __u64 sched_flags;

     /* SCHED_NORMAL, SCHED_BATCH */
     __s32 sched_nice;

     /* SCHED_FIFO, SCHED_RR */
     __u32 sched_priority;

     /* SCHED_DEADLINE (nsec) */
     __u64 sched_runtime;
     __u64 sched_deadline;
     __u64 sched_period;
};

int sched_setattr(pid_t pid,
               const struct sched_attr *attr,
               unsigned int flags)
{
     return syscall(__NR_sched_setattr, pid, attr, flags);
}

int sched_getattr(pid_t pid,
               struct sched_attr *attr,
               unsigned int size,
               unsigned int flags)
{
     return syscall(__NR_sched_getattr, pid, attr, size, flags);
}

void *run_deadline(void *data)
{
     struct sched_attr attr;
     int x = 0;
     int ret;
     unsigned int flags = 0;

     printf("deadline thread started [%ld]\n", gettid());

     attr.size = sizeof(attr);
     attr.sched_flags = 0;
     attr.sched_nice = 0;
     attr.sched_priority = 0;

     /* This creates a 10ms/30ms reservation */
     attr.sched_policy = SCHED_DEADLINE;
     attr.sched_runtime = 10 * 1000 * 1000;
     attr.sched_period = attr.sched_deadline = 30 * 1000 * 1000;

     ret = sched_setattr(0, &attr, flags);
     if (ret < 0) {
             done = 0;
             perror("sched_setattr");
             exit(-1);
     }

     while (!done) {
             x++;
     }

     printf("deadline thread dies [%ld]\n", gettid());
     return NULL;
}

int main (int argc, char **argv)
{
     pthread_t thread;

     printf("main thread [%ld]\n", gettid());

     pthread_create(&thread, NULL, run_deadline, NULL);

     sleep(10);

     done = 1;
     pthread_join(thread, NULL);

     printf("main dies [%ld]\n", gettid());
     return 0;
}


```
