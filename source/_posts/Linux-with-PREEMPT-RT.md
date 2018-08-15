---
title: Linux with PREEMPT_RT
date: 2018-08-15 14:07:25
tags:
---

# Linux with PREEMPT_RT 

## 博主总结

文章摘录自 https://wiki.linuxfoundation.org/realtime/documentation/start 更多内容请前往该网址查看

本博文介绍了如何在标准的Linux系统上**打 PREEMPT_RT 补丁**使之具备实时性，其实就是教你怎么编译内核。

除了介绍如何打补丁外，讲述了如何建立一个基础的**实时线程**，并介绍了实时线程在内存方面的注意事项，包括

1. 锁内存

2. 栈内存

3. 动态内存分配

   ​

   最后给出了在实时线程的基础上给出了**周期任务在实时线程中的实现方式**例子。

## setup Linux with PREEMPT_RT properly

​	Linux in itself is not real time capable. With the additional PREEMPT_RT patch it gains real-time capabilities. The sources have to be downloaded first. After unpacking and patching, the kernel configuration has to be adapted. Then, the kernel can be built and started.

### Getting the sources

​	First, the kernel version should be chosen. After this, take a look if the PREEMPT_RT patch is [available](https://www.kernel.org/pub/linux/kernel/projects/rt) for this particular version.

​	The source of the desired version has to be downloaded (for the Linux kernel as well as for the PREEMPT_RT patch). This example is based on the Linux kernel version 4.4.12.

```bash
$ wget https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.4.12.tar.xz
$ wget https://www.kernel.org/pub/linux/kernel/projects/rt/4.4/patch-4.4.12-rt19.patch.xz
```

After downloading, unpack the archives and patch the Linux kernel:

```bash
$ xz -cd linux-4.4.12.tar.xz | tar xvf -
$ cd linux-4.4.12
$ xzcat ../patch-4.4.12-rt19.patch.xz | patch -p1
```

### Configuring the kernel

The only necessary configuration for real-time Linux kernel is the choice of the “Fully Preemptible Kernel” preemption model (CONFIG_PREEMPT_RT_FULL). All other kernel configuration parameters depend on system requirements. For detailed information about how to configure a kernel have a look at [Linux kernel documentation](https://www.kernel.org/doc/Documentation/kbuild/kconfig.txt).

​	When measuring system latency all kernel debug options should be turned off. They require much overhead and distort the measurement result. Examples for those debug mechanism are:

- DEBUG_PREEMPT

- Lock Debugging (spinlocks, mutexes, etc. . . )

- DEBUG_OBJECTS

- ...

  Some of those debugging mechanisms (like lock debugging) produce a randomized overhead in a range of some micro seconds to several milliseconds depending on the kernel configuration as well as on the compile options (DEBUG_PREEMPT has a low overhead compared to Lock Debugging or DEBUG_OBJECTS).

  However, in the first run of a real-time capable Linux kernel it might be advisable to use those debugging mechanisms. This helps to locate fundamental problems.

### Building the kernel

​	Building the kernel and starting the kernel works similarly to a kernel without PREEMPT_RT patch.

##  Build a simple RT application

​	The POSIX API forms the basis of real-time applications running under PREEMPT_RT. For the real-time thread a POSIX thread is used (pthread). Every real-time application needs proper handling in several basic areas like scheduling, priority, memory locking and stack prefaulting.

### Basic prerequisites

​	Three basic prerequisites are introduced in the next subsections, followed by a short example illustrating those aspects.

#### Scheduling and priority

​	The [scheduling policy](https://wiki.linuxfoundation.org/realtime/documentation/technical_basics/sched_policy_prio/start) as well as the priority must be set by the application explicitly. There are two possibilities for this:

1. **Using sched_setscheduler()**

   This funcion needs to be called in the start routine of the pthread before calculating RT specific stuff.

2. **Using pthread attributes** 

   The functions `pthread_attr_setschedpolicy()` and `pthread_attr_setschedparam()`offer the interfaces to set policy and priority. Furthermore scheduler inheritance needs to be set properly to PTHREAD_EXPLICIT_SCHED by using `pthread_attr_setinheritsched()`. This forces the new thread to use the policy and priority specified by the pthread attributes and not to use the inherit scheduling of the thread which created the real-time thread.

### Memory for Real-time Applications

​	Proper handling of memory will improve a real-time application's deterministic behavior. Three areas of memory management within the purview of a real-time application are considered :

1. **Memory Locking**
2. **Stack Memory for RT threads**
3. **Dynamic memory allocation**

Keep in mind that the [usual sequence](https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/application_base) is for an application to begin its execution as a regular (non-RT) application, then create the RT threads with appropriate resources and scheduling parameters.

#### Memory Locking

Memory locking APIs allow an application to instruct the kernel to associate (some or all of its) virtual memory pages with real page frames and keep it that way. In other words :

- Memory locking APIs will trigger the necessary page-faults, to bring in the pages being locked, to physical memory. Consequently first access to a locked-memory (following an `mlock*()` call) will already have physical memory assigned and will not page fault (in RT-critical path). This removes the need to explicitly pre-fault these memory.

- Further memory locking prevents an application's memory pages, from being paged-out, anytime during its lifetime even in when the overall system is facing memory pressure.

Applications can either use `mlock(…)` or `mlockall(…)` for memory locking. Specifics of these C Library calls can be found here [The GNU C Library: Locking pages](http://www.gnu.org/software/libc/manual/html_node/Locking-Pages.html). Note that these calls requires the application to have sufficient privileges (i.e. [CAP_IPC_LOCK capability](http://man7.org/linux/man-pages/man7/capabilities.7.html)) to succeed.

While `mlock(<addr>, <length>)` locks specific pages (described by *address* and *length*), `mlockall(…)` locks an application's entire virtual address space (i.e [globals](https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/memory/mlockall_globals_sample), stack, heap, code) in physical memory. The trade-off between convenience and locking-up excess RAM should drive the choice of one over the other. Locking only those areas which are accessed by RT-threads (using `mlock(…)`) could be cheaper than blindly using `mlockall(…)` which will end-up locking all memory pages of the application (i.e. even those which are used only by non-RT threads).

The snippet below illustrates the usage of `mlockall(…)` :

```c
  /* Lock all current and future pages from preventing of being paged to swap */
  if (mlockall( MCL_CURRENT | MCL_FUTURE )) { 
          perror("mlockall failed");
          /* exit(-1) or do error handling */
  }
```

​	Real-time applications should use memory-locking APIs early in their life, prior to performing real-time activities, so as to not incur page-faults in RT critical path. Failing to do so may significantly impact the determinism of the application.

​	Note that memory locking is required irrespective of whether *swap area* is configured for a system or not. This is because pages for read-only memory areas (like program code) could be dropped from the memory, when the system is facing memory pressure. Such read-only pages (being identical to on-disk copy), would be brought back straight from the disk (and not swap), resulting in page-faults even on setups without a swap-memory.

#### Stack Memory for RT threads

​	All threads (RT and non-RT) within an application have their own private stack. It is recommended that an application should understand the *stack size* needs for its RT threads and set them explicitly before spawning them. This can be done via the `pthread_attr_setstacksize(…)` call as shown in the snippet below. If the size is not explicitly set, then the thread gets the default stack size (`pthread_attr_getstacksize()` can be used to find out how much this is, it was 8MB at the time of this writing).

​	Aforementioned `mlockall(…)` is sufficient to pin the entire thread stack in RAM, so that pagefaults are not incurred while the thread stack is being used. If the application spawns a large number of RT threads, it is advisable to specify a smaller stack size (than the default) in the interest of not exhausting memory.

```c
 static void create_rt_thread(void)
 {
         pthread_t thread;
         pthread_attr_t attr;
 
         /* init to default values */
         if (pthread_attr_init(&attr))
   	         error(1);
         /* Set a specific stack size   */
         if (pthread_attr_setstacksize(&attr, PTHREAD_STACK_MIN + MY_STACK_SIZE))
   	        error(2);
         /* And finally start the actual thread */
         pthread_create(&thread, &attr, rt_func, NULL);
 }
```

​	Details: The entire stack of every thread inside the application is forced to RAM when `mlockall(MCL_CURRENT)` is called. Threads created after a call to `mlockall(MCL_CURRENT | MCL_FUTURE)` will generate page faults immediately (on creation), as the new stack is immediately forced to RAM (due to the MCL_FUTURE flag). So all RT threads need to be created at startup time, before the RT show time. With `mlockall(…)` no explicit additional prefaulting necessary to avoid pagefaults during first (or subsequent) access.

#### Dynamic memory allocation in RT threads

​	Real-time threads should avoid doing dynamic memory allocation / freeing while in RT critical path. The suggested recommendation for real-time threads, is to do the allocations, prior-to entering RT critical path. Subsequently RT threads, within their RT-critical path, can use this pre-allocated dynamic memory, provided that it is locked as described [here](https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/memory#memory-locking).

​	Non RT-threads within the applications have no restrictions on dynamic allocation / free.

### Example

```C
/*                                                                  
 * POSIX Real Time Example
 * using a single pthread as RT thread
 */
 
#include <limits.h>
#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
 
void *thread_func(void *data)
{
        /* Do RT specific stuff here */
        return NULL;
}
 
int main(int argc, char* argv[])
{
        struct sched_param param;
        pthread_attr_t attr;
        pthread_t thread;
        int ret;
 
        /* Lock memory */
        if(mlockall(MCL_CURRENT|MCL_FUTURE) == -1) {
                printf("mlockall failed: %m\n");
                exit(-2);
        }
 
        /* Initialize pthread attributes (default values) */
        ret = pthread_attr_init(&attr);
        if (ret) {
                printf("init pthread attributes failed\n");
                goto out;
        }
 
        /* Set a specific stack size  */
        ret = pthread_attr_setstacksize(&attr, PTHREAD_STACK_MIN);
        if (ret) {
            printf("pthread setstacksize failed\n");
            goto out;
        }
 
        /* Set scheduler policy and priority of pthread */
        ret = pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
        if (ret) {
                printf("pthread setschedpolicy failed\n");
                goto out;
        }
        param.sched_priority = 80;
        ret = pthread_attr_setschedparam(&attr, &param);
        if (ret) {
                printf("pthread setschedparam failed\n");
                goto out;
        }
        /* Use scheduling parameters of attr */
        ret = pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
        if (ret) {
                printf("pthread setinheritsched failed\n");
                goto out;
        }
 
        /* Create a pthread with specified attributes */
        ret = pthread_create(&thread, &attr, thread_func, NULL);
        if (ret) {
                printf("create pthread failed\n");
                goto out;
        }
 
        /* Join the thread and wait until it is done */
        ret = pthread_join(thread, NULL);
        if (ret)
                printf("join pthread failed: %m\n");
 
out:
        return ret;
}
```



## Build a basic cyclic application

#### Cyclic Task

​	A cyclic task is one which is repeated after a fixed period of time like reading sensor data every 100 ms. The execution time for the cyclic task should always be less than the period of the task. Following are the mechanisms which we will be looking at for implementing cyclic task:

- nanosleep
- EDF Scheduling

#### Current Time

​	There are multiple ways to get current time – gettimeofday, time, clock_gettime, and some other processor specific implementations. Some of them, like gettimeofday, will get time from the system clock. The system clock can be modified by other processes. Which means that the clock can go back in time. clock_gettime with CLOCK_MONOTONIC clock can be used to avoid this problem. CLOCK_MONOTONIC argument ensures that we get a nonsettable monotonically increasing clock that measures time from some unspecified point in the past that does not change after system startup[1]. It is also important to ensure we do not waste a lot of CPU cycles to get the current time. CPU specific implementations to get the current time will be helpful here.

#### Basic Stub

Any mechanism for implementing a cyclic task can be divided into the following parts:

- periodic_task_init(): Initialization code for doing things like requesting timers, initializing variables, setting timer periods.
- do_rt_task(): The real time task is done here.
- wait_rest_of_period(): After the task is done, wait for the rest of the period. The assumption here is the task requires less time to complete compared to the period length.
- struct period_info: This is a struct which will be used to pass around data required by the above mentioned functions.

The stub for the real time task will look like:

```c
void *simple_cyclic_task(void *data)
{
        struct period_info pinfo;
 
        periodic_task_init(&pinfo);
 
        while (1) {
                do_rt_task();
                wait_rest_of_period(&pinfo);
        }
 
        return NULL;
}
```

#### Examples

**clock_nanosleep**

clock_nanosleep() is used to ask the process to sleep for certain amount of time. nanosleep() can also be used to sleep. But, nanosleep() uses CLOCK_REALTIME which can be changed by another processes and hence can be discontinuous or jump back in time. In clock_nanosleep, CLOCK_MONOTONIC is explicitly specified. This is a immutable clock which does not change after startup. The periodicity is achieved by using absolute time to specify the end of each period. More information on clock_nanosleep at <http://man7.org/linux/man-pages/man2/clock_nanosleep.2.html>

```c
struct period_info {
        struct timespec next_period;
        long period_ns;
};
 
static void inc_period(struct period_info *pinfo) 
{
        pinfo->next_period.tv_nsec += pinfo->period_ns;
 
        while (pinfo->next_period.tv_nsec >= 1000000000) {
                /* timespec nsec overflow */
                pinfo->next_period.tv_sec++;
                pinfo->next_period.tv_nsec -= 1000000000;
        }
}
 
static void periodic_task_init(struct period_info *pinfo)
{
        /* for simplicity, hardcoding a 1ms period */
        pinfo->period_ns = 1000000;
 
        clock_gettime(CLOCK_MONOTONIC, &(pinfo->next_period));
}
 
static void do_rt_task()
{
        /* Do RT stuff here. */
}
 
static void wait_rest_of_period(struct period_info *pinfo)
{
        inc_period(pinfo);
 
        /* for simplicity, ignoring possibilities of signal wakes */
        clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &pinfo->next_period, NULL);
}
```

#### EDF Scheduler

​	Recently, earliest deadline first scheduling algorithm has been merged in the mainline kernel. Now, users can specify runtime, period and deadline of a task and they scheduler will run the task every specified period and will make sure the deadline is met. The scheduler will also let user know if the tasks(or a set of tasks) cannot be scheduled because the deadline won't be met.

​	More information about the EDF scheduler including an example of implementation can be found at: <https://www.kernel.org/doc/Documentation/scheduler/sched-deadline.txt>