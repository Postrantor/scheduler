---
tip: translate by baidu@2023-09-05 16:03:25
url: https://lwn.net/Articles/743740/
title: Deadline scheduling part 1 — overview and theory [LWN.net] --- 截止日期安排第 1 部分 — 概述和理论 [LWN.net]
date: 2023-09-05 16:00:18
tag:
summary: The deadline scheduler enables the user to specify a realtime task's
> 摘要：截止日期调度程序允许用户指定实时任务的
---

## using well-defined......

**Please consider subscribing to LWN**

Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](https://lwn.net/subscribe/) to join up and keep LWN on the net.

> 订阅是 LWN.net 的生命线。如果你喜欢这些内容并想看更多，你的订阅将有助于确保 LWN 继续蓬勃发展。请访问[本页](https://lwn.net/subscribe/)加入并保持 LWN 在网络上。

Realtime systems are computing systems that must react within precise time constraints to events. In such systems, the correct behavior does not depend only on the logical behavior, but also in the timing behavior. In other words, the response for a request is only correct if the logical result is correct and produced within a deadline. If the system fails to provide the response within the deadline, the system is showing a defect. In a multitasking operating system, such as Linux, a realtime scheduler is responsible for coordinating the access to the CPU, to ensure that all realtime tasks in the system accomplish their job within the deadline.

> 实时系统是必须在精确的时间约束内对事件做出反应的计算系统。在这样的系统中，正确的行为不仅取决于逻辑行为，还取决于时序行为。换句话说，只有在逻辑结果正确并且在截止日期内产生的情况下，对请求的响应才是正确的。如果系统未能在截止日期内提供响应，则表明系统存在缺陷。在 Linux 等多任务操作系统中，实时调度器负责协调对 CPU 的访问，以确保系统中的所有实时任务在截止日期内完成任务。

The deadline scheduler enables the user to specify the tasks' requirements using well-defined realtime abstractions, allowing the system to make the best scheduling decisions, guaranteeing the scheduling of realtime tasks even in higher-load systems.

> 截止日期调度器使用户能够使用定义良好的实时抽象来指定任务的要求，使系统能够做出最佳的调度决策，即使在负载较高的系统中也能保证实时任务的调度。

This article provides an introduction to realtime scheduling and some of the theory behind it. The second installment will be dedicated to the Linux deadline scheduler in particular.

> 本文介绍了实时调度及其背后的一些理论。第二部分将专门介绍 Linux 截止日期调度程序。

#### Realtime schedulers in Linux

Realtime tasks differ from non-realtime tasks by the constraint of having to produce a response for an event within a deadline. To schedule a realtime task to accomplish its timing requirements, Linux provides two realtime schedulers: the POSIX realtime scheduler (henceforth called the "realtime scheduler") and the deadline scheduler.

> 实时任务与非实时任务的不同之处在于必须在截止日期内为事件生成响应。为了调度实时任务以满足其时序要求，Linux 提供了两个实时调度器：POSIX 实时调度器（此后称为“实时调度器”）和截止日期调度器。

The POSIX realtime scheduler, which provides the FIFO (first-in-first-out) and RR (round-robin) scheduling policies, schedules each task according to its fixed priority. The task with the highest priority will be served first. In realtime theory, this scheduler is classified as a fixed-priority scheduler. The difference between the FIFO and RR schedulers can be seen when two tasks share the same priority. In the FIFO scheduler, the task that arrived first will receive the processor, running until it goes to sleep. In the RR scheduler, the tasks with the same priority will share the processor in a round-robin fashion. Once an RR task starts to run, it will run for a maximum quantum of time. If the task does not block before the end of that time slice, the scheduler will put the task at the end of the round-robin queue of the tasks with the same priority and select the next task to run.

> POSIX 实时调度器提供 FIFO（先进先出）和 RR（循环）调度策略，根据每个任务的固定优先级对其进行调度。优先级最高的任务将首先得到服务。在实时理论中，该调度器被归类为固定优先级调度器。当两个任务共享相同的优先级时，可以看出 FIFO 和 RR 调度器之间的差异。在 FIFO 调度程序中，最先到达的任务将接收处理器，一直运行到进入睡眠状态。在 RR 调度器中，具有相同优先级的任务将以循环方式共享处理器。一旦 RR 任务开始运行，它将运行最长的时间。如果任务在该时间片结束前没有阻塞，调度程序将把该任务放在具有相同优先级的任务的循环队列的末尾，并选择下一个要运行的任务。

In contrast, the deadline scheduler, as its name says, schedules each task according to the task's deadline. The task with the earliest deadline will be served first. Each scheduler requires a different setup for realtime tasks. In the realtime scheduler, the user needs to provide the scheduling policy and the fixed priority. For example:

> 相反，正如它的名字所说，截止日期调度器根据任务的截止日期来调度每个任务。最后期限最早的任务将首先送达。对于实时任务，每个调度程序都需要不同的设置。在实时调度器中，用户需要提供调度策略和固定优先级。例如：

```
    chrt -f 10 video_processing_tool
```

With this command, the video_processing_tool task will be scheduled by the realtime scheduler, with a priority of 10, under the FIFO policy (as requested by the -f flag).

> 有了这个命令，视频\_processing_tool 任务将由实时调度器在 FIFO 策略下调度，优先级为 10（根据-f 标志的请求）。

In the deadline scheduler, instead, the user has three parameters to set: the period, the run time, and the deadline. The period is the activation pattern of the realtime task. In a practical example, if a video-processing task must process 60 frames per second, a new frame will arrive every 16 milliseconds, so the period is 16 milliseconds.

> 相反，在截止日期调度程序中，用户有三个参数要设置：周期、运行时间和截止日期。周期是实时任务的激活模式。在一个实际示例中，如果视频处理任务必须每秒处理 60 帧，则每 16 毫秒就会有一个新帧到达，因此周期为 16 毫秒。

The run time is the amount of CPU time that the application needs to produce the output. In the most conservative case, the runtime must be the worst-case execution time (WCET), which is the maximum amount of time the task needs to process one period's worth of work. For example, a video processing tool may take, in the worst case, five milliseconds to process the image. Hence its run time is five milliseconds.

> 运行时间是应用程序生成输出所需的 CPU 时间。在最保守的情况下，运行时必须是最坏情况执行时间（WCET），这是任务处理一段时间的工作所需的最长时间。例如，在最坏的情况下，视频处理工具可能需要 5 毫秒来处理图像。因此，它的运行时间是 5 毫秒。

The deadline is the maximum time in which the result must be delivered by the task, relative to the period. For example, if the task needs to deliver the processed frame within ten milliseconds, the deadline will be ten milliseconds.

> 截止日期是任务必须交付结果的最长时间，相对于时间段而言。例如，如果任务需要在 10 毫秒内交付处理后的帧，则截止日期为 10 毫秒。

It is possible to set deadline scheduling parameters using the chrt command. For example, the above-mentioned tool could be started with the following command:

> 可以使用 chrt 命令设置截止日期调度参数。例如，可以使用以下命令启动上述工具：

```
    chrt -d --sched-runtime 5000000 --sched-deadline 10000000 \
    	    --sched-period 16666666 0 video_processing_tool
```

Where:

- --sched-runtime 5000000 is the run time specified in nanoseconds

- --sched-deadline 10000000 is the relative deadline specified in nanoseconds.

  > \*--sched deadline 10000000 是以纳秒为单位指定的相对截止时间。

- --sched-period 16666666 is the period specified in nanoseconds

- 0 is a placeholder for the (unused) priority, required by the chrt command
  > \*0 是 chrt 命令所需的（未使用的）优先级的占位符

In this way, the task will have a guarantee of 5ms of CPU time every 16.6ms, and all of that CPU time will be available for the task before the 10ms deadline passes.

> 通过这种方式，任务将保证每 16.6ms 有 5ms 的 CPU 时间，并且在 10ms 截止日期之前，所有 CPU 时间都将可用于任务。

Although the deadline scheduler's configuration looks complex, it is not. By giving the correct parameters, which are only dependent on the application itself, the user does not need to be aware of all the other tasks in the system to be sure that the application will deliver its results before the deadline. When using the realtime scheduler, instead, the user must take into account all of the system's tasks to be able to define which is the correct fixed priority for any task.

> 尽管截止日期调度程序的配置看起来很复杂，但事实并非如此。通过给出仅取决于应用程序本身的正确参数，用户不需要知道系统中的所有其他任务，即可确保应用程序将在截止日期前交付其结果。相反，当使用实时调度程序时，用户必须考虑系统的所有任务，才能定义任何任务的正确固定优先级。

Since the deadline scheduler knows how much CPU each deadline task will need, it knows when the system can (or cannot) admit new tasks. So, rather than allowing the user to overload the system, the deadline scheduler denies the addition of more deadline tasks, guaranteeing that all deadline tasks will have CPU time to accomplish their tasks with, at least, a bounded tardiness.

> 由于截止日期调度程序知道每个截止日期任务需要多少 CPU，因此它知道系统何时可以（或不能）接纳新任务。因此，截止日期调度器不允许用户使系统过载，而是拒绝添加更多的截止日期任务，保证所有截止日期任务都有 CPU 时间来完成任务，至少有一定的延迟。

In order to further discuss benefits of the deadline scheduler it is necessary to take a step back and look at the bigger picture. To that end, the next section explains a little bit about realtime scheduling theory.

> 为了进一步讨论截止日期调度程序的好处，有必要退一步，着眼于大局。为此，下一节将稍微解释一下实时调度理论。

#### A realtime scheduling overview

In scheduling theory, realtime schedulers are evaluated by their ability to schedule a set of tasks while meeting the timing requirements of all realtime tasks. In order to provide deterministic response times, realtime tasks must have a deterministic timing behavior. The task model describes the deterministic behavior of a task.

> 在调度理论中，实时调度器通过其在满足所有实时任务的时间要求的同时调度一组任务的能力来评估。为了提供确定性的响应时间，实时任务必须具有确定性的时序行为。任务模型描述了任务的确定性行为。

Each realtime task is composed of _N_ recurrent activations; a task activation is known as a **job**. A task is said to be **periodic** when a job takes place after a fixed offset of time from its previous activation. For instance, a periodic task with period of 2ms will be activated every 2ms. Tasks can also be **sporadic**. A sporadic task is activated after, at least, a minimum inter-arrival time from its previous activation. For instance, a sporadic task with a 2ms period will be activated after at least 2ms from the previous activation. Finally, a task can be **aperiodic**, when there is no activation pattern that can be established.

> 每个实时任务由*N*个循环激活组成；任务激活被称为**作业**。当一个作业在距上次激活有固定时间偏移后发生时，任务被称为**周期性**。例如，周期为 2ms 的周期性任务将每 2ms 激活一次。任务也可以是**零星的**。零星任务至少在其先前激活的最短到达间隔时间之后被激活。例如，周期为 2ms 的零星任务将在上次激活后至少 2ms 后被激活。最后，当没有可以建立的激活模式时，任务可以是**非周期性**。

Tasks can have an **implicit deadline**, when the deadline is equal to the activation period, or a **constrained deadline**, when the deadline can be less than (or equal to) the period. Finally, a task can have an **arbitrary deadline**, where the deadline is unrelated to the period.

> 当截止日期等于激活期限时，任务可以具有**隐含截止日期**，或者当截止日期可以小于（或等于）激活期限时具有**约束截止日期**。最后，一个任务可以有一个**任意的截止日期**，其中截止日期与周期无关。

Using these patterns, realtime researchers have developed ways to compare scheduling algorithms by their ability to schedule a given task set. It turns out that, for uniprocessor systems, the Early Deadline First (EDF) scheduler was found to be optimal. A scheduling algorithm is optimal when it fails to schedule a task set only when no other scheduler can schedule it. The deadline scheduler is optimal for periodic and sporadic tasks with deadlines less than or equal to their periods on uniprocessor systems. Actually, for either periodic or sporadic tasks with implicit deadlines, the EDF scheduler can schedule any task set as long as the task set does not use more than 100% of the CPU time. The Linux deadline scheduler implements the EDF algorithm.

> 利用这些模式，实时研究人员开发了通过调度给定任务集的能力来比较调度算法的方法。结果表明，对于单处理器系统，早期截止日期优先（EDF）调度器是最优的。当调度算法只有在没有其他调度程序可以调度任务集的情况下才无法调度任务集时，它是最优的。对于单处理器系统上截止日期小于或等于其周期的周期性和偶发性任务，截止日期调度程序是最佳的。实际上，对于具有隐含截止日期的周期性或偶发性任务，EDF 调度器可以调度任何任务集，只要该任务集使用的 CPU 时间不超过 100%。Linux 截止日期调度程序实现 EDF 算法。

Consider, for instance, a system with three periodic tasks with deadlines equal to their periods:

> 例如，考虑一个系统，该系统有三个周期性任务，其截止日期等于它们的周期：

<table><tbody><tr><th>Task</th><th>Runtime<br>(WCET)</th><th>Period</th></tr><tr><td>T<sub>1</sub></td><td>1</td><td>4</td></tr><tr><td>T<sub>2</sub></td><td>2</td><td>6</td></tr><tr><td>T<sub>3</sub></td><td>3</td><td>8</td></tr></tbody></table>

The CPU time utilization (U) of this task set is less than 100%:

```
    U =  1/4 + 2/6 + 3/8 = 23/24
```

For such a task set, the EDF scheduler would present the following behavior:

> 对于这样的任务集，EDF 调度器将呈现以下行为：

![](https://static.lwn.net/images/2018/deadline/dl.png)

However, it is not possible to use a fixed-priority scheduler to schedule this task set while meeting every deadline; regardless of the assignment of priorities, one task will not run in time to get its work done. The resulting behavior will look like this:

> 但是，不可能在满足每个截止日期的同时使用固定优先级的调度器来调度此任务集；不管优先级的分配如何，一个任务都无法及时完成任务。由此产生的行为如下所示：

![](https://static.lwn.net/images/2018/deadline/fp.png)

The main advantage of deadline scheduling is that, once you know each task's parameters, you do not need to analyze all of the other tasks to know that your tasks will all meet their deadlines. Deadline scheduling often results in fewer context switches and, on uniprocessor systems, deadline scheduling is able to schedule more tasks than fixed priority-scheduling while meeting every task's deadline. However, the deadline scheduler also has some disadvantages.

> 截止日期安排的主要优点是，一旦你知道了每个任务的参数，你就不需要分析所有其他任务来知道你的任务都会在截止日期前完成。最后期限调度通常导致更少的上下文切换，并且在单处理器系统上，在满足每个任务的最后期限的同时，最后期限调度能够比固定优先级调度调度调度更多的任务。然而，截止日期调度器也有一些缺点。

The deadline scheduler provides a guarantee of accomplishing each task's deadline, but it is not possible to ensure a minimum response time for any given task. In the fixed-priority scheduler, the highest-priority task always has the minimum response time, but that is not possible to guarantee with the deadline scheduler. The EDF scheduling algorithm is also more complex than fixed-priority, which can be implemented with O(1) complexity. In contrast, the deadline scheduler is O(log(n)). However, the fixed-priority requires an “offline computation” of the best set of priorities by the user, which can be as complex as O(N!).

> 最后期限调度器提供了完成每个任务的最后期限的保证，但不可能确保任何给定任务的最短响应时间。在固定优先级调度器中，最高优先级的任务总是具有最小的响应时间，但这在最后期限调度器中是不可能保证的。EDF 调度算法也比固定优先级更复杂，固定优先级可以用 O（1）复杂性来实现。相反，截止日期调度器是 O（log（n））。然而，固定优先级需要用户对最佳优先级集进行“离线计算”，这可能与 O（N！）一样复杂。

If, for some reason, the system becomes overloaded, for instance due to the addition of a new task or a wrong WCET estimation, it is possible to face a domino effect: once one task misses its deadline by running for more than its declared run time, all other tasks may miss their deadlines as shown by the regions in red below:

> 如果由于某种原因，系统过载，例如由于添加了一个新任务或 WCET 估计错误，则可能会面临多米诺骨牌效应：一旦一个任务因运行时间超过其声明的运行时间而错过了截止日期，则所有其他任务都可能错过截止日期，如下面红色区域所示：

![](https://static.lwn.net/images/2018/deadline/domino.png)

In contrast, with fixed-priority scheduling, only the tasks with lower priority than the task which missed the deadline will be affected.

> 相反，在固定优先级调度的情况下，只有优先级低于错过截止日期的任务才会受到影响。

In addition to the prioritization problem, multi-core systems add an allocation problem. On a multi-core system, the scheduler also needs to decide where the tasks can run. Generally, the scheduler can be classified as one of the following:

> 除了优先级问题之外，多核系统还增加了分配问题。在多核系统上，调度器还需要决定任务可以在哪里运行。通常，调度器可以分为以下几种：

![](https://static.lwn.net/images/2018/deadline/schedtypes.png)

- **Global**: When a single scheduler manages all M CPUs of the system. In other words, tasks can migrate to all CPUs.

> **\*全局**：当单个调度器管理系统的所有 M 个 CPU 时。换句话说，任务可以迁移到所有 CPU。

- **Clustered**: When a single scheduler manages a disjoint subset of the M CPUs. In other words, tasks can migrate to just a subset of the available CPUs.

> **\*集群**：当单个调度程序管理 M 个 CPU 的不相交子集时。换句话说，任务可以迁移到可用 CPU 的一个子集。

- **Partitioned**: When each scheduler manages a single CPU, so no migration is allowed.

> **\*分区**：当每个调度程序管理一个 CPU 时，不允许迁移。

- **Arbitrary**: Each task can run on an arbitrary set of CPUs.

In multi-core systems, global, clustered, and arbitrary deadline schedulers are not optimal. The theory for multi-core scheduling is more complex than for single-core systems due to many anomalies. For example, in a system with M processors, it is possible to schedule M tasks with a run time equal to the period. For instance, a system with four processors can schedule four "BIG" tasks with both run time and period equal to 1000ms. In this case, the system will reach the maximum utilization of:

> 在多核系统中，全局、集群和任意截止日期调度器不是最优的。由于许多异常情况，多核调度理论比单核系统更复杂。例如，在具有 M 个处理器的系统中，可以调度 M 个任务，其运行时间等于周期。例如，一个有四个处理器的系统可以调度四个“大”任务，运行时间和周期都等于 1000ms。在这种情况下，系统将达到以下各项的最大利用率：

```
    4 * 1000/1000 = 4


```

The resulting scheduling behavior will look like:

![](https://static.lwn.net/images/2018/deadline/bigs.png)

It is intuitive to think that a system with a lower load will be schedulable too, as it is for single-processor systems. For example, in a system with four processors, a task set composed of four small tasks with the minimum runtime, let's say 1ms, at every 999 milliseconds period, and just one task BIG task, with runtime and period of one second. The load of this system is:

> 直观地认为，负载较低的系统也可以调度，就像单处理器系统一样。例如，在一个有四个处理器的系统中，一个任务集由四个运行时间最小的小任务组成，比如说，每 999 毫秒运行 1 毫秒，而只有一个任务 BIG 任务，运行时间和周期为 1 秒。该系统的负载为：

```
    4 * (1/999) + 1000/1000 = 1.004


```

As 1.004 is smaller than four, intuitively, one might say that the system is schedulable, But that is not true for global EDF scheduling. That is because, if all tasks are released at the same time, the M small tasks will be scheduled in the M available processors. Then, the big task will be able to start only after the small tasks have run, hence finishing its computation after its deadline. As illustrated below. This is known as the Dhall's effect.

> 由于 1.004 小于 4，直观地说，可以说系统是可调度的，但对于全局 EDF 调度来说，情况并非如此。这是因为，如果同时发布所有任务，则 M 个小任务将在 M 个可用处理器中进行调度。然后，大任务只有在小任务运行后才能启动，从而在截止日期后完成计算。如下图所示。这就是所谓的达尔效应。

![](https://static.lwn.net/images/2018/deadline/dhall.png)

Distribution of tasks to processors turns out to be an NP-hard problem (a bin-packing problem, essentially) and, due to other anomalies, there is no dominance of one scheduling algorithm over any others.

> 任务到处理器的分配被证明是一个 NP 难问题（本质上是一个装箱问题），并且由于其他异常，一种调度算法不比任何其他调度算法占主导地位。

With this background in place, we can turn to the details of the Linux deadline scheduler and the best ways to take advantage of its capabilities while avoiding the potential problems. See [the second half of this series](https://lwn.net/Articles/743946/), to be published soon, for the full story.

> 有了这个背景，我们可以了解 Linux 截止日期调度程序的细节，以及在避免潜在问题的同时利用其功能的最佳方法。参见[本系列的后半部分](https://lwn.net/Articles/743946/)，即将出版，以获取完整的故事。

<table><tbody><tr><th colspan="2">Index entries for this article</th></tr><tr><td><a href="https://lwn.net/Kernel/Index">Kernel</a></td><td><a href="https://lwn.net/Kernel/Index#Realtime-Deadline_scheduling">Realtime/Deadline scheduling</a></td></tr><tr><td><a href="https://lwn.net/Kernel/Index">Kernel</a></td><td><a href="https://lwn.net/Kernel/Index#Scheduler-Deadline_scheduling">Scheduler/Deadline scheduling</a></td></tr><tr><td><a href="https://lwn.net/Archives/GuestIndex/">GuestArticles</a></td><td><a href="https://lwn.net/Archives/GuestIndex/#Bristot_de_Oliveira_Daniel">Bristot de Oliveira, Daniel</a></td></tr></tbody></table>

([Log in](https://lwn.net/Login/?target=/Articles/743740/) to post comments)

> （[登录](https://lwn.net/Login/?target=/Articles/743740/)发布评论）
