## 1. 实验目的

- 掌握 `Linux` 下的多进程编程技术；
- 通过对进程运行轨迹的跟踪来形象化进程的概念；
- 在进程运行轨迹跟踪的基础上进行相应的数据统计，从而能对进程调度算法进行实际的量化评价， 更进一步加深对调度和调度算法的理解，获得能在实际操作系统上对调度算法进行实验数据对比的直接经验。

## 2. 实验内容

进程从创建（ `Linux` 下调用 `fork()` ）到结束的整个过程就是进程的生命期， 进程在其生命期中的运行轨迹实际上就表现为进程状态的多次切换，如进程创建以后会成为就绪态； 当该进程被调度以后会切换到运行态；在运行的过程中如果启动了一个文件读写操作， 操作系统会将该进程切换到阻塞态（等待态）从而让出 `CPU` ； 当文件读写完毕以后，操作系统会在将其切换成就绪态，等待进程调度算法来调度该进程执行……

本次实验包括如下内容：

1. 基于模板 `process.c` 编写多进程的样本程序，实现如下功能：
   - 所有子进程都并行运行，每个子进程的实际运行时间一般不超过 `30` 秒；
   - 父进程向标准输出打印所有子进程的 `id` ，并在所有子进程都退出后才退出；
2. 在 `Linux 0.11` 上实现进程运行轨迹的跟踪。基本任务是在内核中维护一个日志文件 `/var/process.log` ，把从操作系统启动到系统关机过程中所有进程的运行轨迹都记录在这一 `log` 文件中。
3. 在修改过的 `0.11` 上运行样本程序，通过分析 `log` 文件，统计该程序建立的所有进程的等待时间、完成时间（周转时间）和运行时间，然后计算平均等待时间，平均完成时间和吞吐量。可以自己编写统计程序，也可以使用 `python` 脚本程序 `stat_log.py` 进行统计。
4. 修改 `0.11` 进程调度的时间片，然后再运行同样的样本程序，统计同样的时间数据，和原有的情况对比，体会不同时间片带来的差异。

`/var/process.log` 文件的格式必须为：

```text
pid X time
```

其中：

- `pid` 是进程的 `ID` ；
- `X` 可以是 `N` , `J` , `R` , `W` 和 `E` 中的任意一个，分别表示进程新建( `N` )、进入就绪态( `J` )、进入运行态( `R` )、进入阻塞态( `W` )和退出( `E` )；
- `time` 表示 `X` 发生的时间。这个时间不是物理时间，而是系统的滴答时间( `tick` )；

三个字段之间用制表符分隔。 例如：

```text
12    N    1056
12    J    1057
4     W    1057
12    R    1057
13    N    1058
13    J    1059
14    N    1059
14    J    1060
15    N    1060
15    J    1061
12    W    1061
15    R    1061
15    J    1076
14    R    1076
14    E    1076
……
```

## 3. 实验过程

### 3.1. 基于模板 `process.c` 编写多进程的样本程序

`process.c` 是样本程序的模板。它主要实现了一个函数：

```C
/*
* 此函数按照参数占用CPU和I/O时间
* last: 函数实际占用CPU和I/O的总时间，不含在就绪队列中的时间，>=0是必须的
* cpu_time: 一次连续占用CPU的时间，>=0是必须的
* io_time: 一次I/O消耗的时间，>=0是必须的
* 如果last > cpu_time + io_time，则往复多次占用CPU和I/O，直到总运行时间超过last为止
* 所有时间的单位为秒
*/
cpuio_bound(int last, int cpu_time, int io_time);

//比如一个进程如果要占用10秒的CPU时间，它可以调用：
cpuio_bound(10, 1, 0);  // 只要cpu_time>0，io_time=0，效果相同


// 以I/O为主要任务：
cpuio_bound(10, 0, 1);  // 只要cpu_time=0，io_time>0，效果相同

// CPU和I/O各1秒钟轮回：
cpuio_bound(10, 1, 1);

// 较多的I/O，较少的CPU：
cpuio_bound(10, 1, 9);  // I/O时间是CPU时间的9倍
```

修改此模板，用 `fork()` 建立若干个同时运行的子进程，父进程等待所有子进程退出后才退出，每个子进程按照意愿做不同或相同的 `cpuio_bound()` ，从而完成一个个性化的样本程序。

```c
#include <stdio.h>
#include <sys/times.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <time.h>
#include <unistd.h>

#define HZ 100
#define NUM_CHILD 5 /* 子进程数量 */

void cpuio_bound(int last, int cpu_time, int io_time);

int main(int argc, char *argv[])
{
    pid_t pid[NUM_CHILD]; /* 子进程 PID */
    int exit_pid = 0;

    printf("Parent PID is: %d\n", getpid());
    /* 创建 NUM_CHILD 个子进程 */
    for (int i = 0; i < NUM_CHILD; i++)
    {
        pid[i] = fork(); /* 创建子进程 */
        if (pid[i] < 0)
        { /* 创建失败 */
            printf("Failed to fork child process %d \n", i + 1);
            exit(1);
        }
        if (pid[i] == 0)
        { /* 子进程创建成功  */
            if (i % 3 == 0)
            {
                printf("%d: The child process has been created. PID is %d).\n",
                       i, getpid());
                cpuio_bound(10, 1, 0);
            }
            else if (i % 3 == 1)
            {
                printf("%d: The child process has been created. PID is %d).\n",
                       i, getpid());
                cpuio_bound(10, 1, 9);
            }
            else
            {
                printf("%d: The child process has been created. PID is %d).\n",
                       i, getpid());
                cpuio_bound(10, 1, 1);
            }
            exit(0); /* 子进程结束 */
        }
    }

    /* 父进程等待所有子进程结束 */
    for (int i = 0; i < NUM_CHILD; i++)
    {
        exit_pid = wait(NULL);
        printf("%d have exited.\n", exit_pid);
    }

    printf("The program was executed successfully.\n");
    return 0;
}

/*
 * 此函数按照参数占用CPU和I/O时间
 * last: 函数实际占用CPU和I/O的总时间，不含在就绪队列中的时间，>=0是必须的
 * cpu_time: 一次连续占用CPU的时间，>=0是必须的
 * io_time: 一次I/O消耗的时间，>=0是必须的
 * 如果last > cpu_time + io_time，则往复多次占用CPU和I/O
 * 所有时间的单位为秒
 */
void cpuio_bound(int last, int cpu_time, int io_time)
{
    struct tms start_time, current_time;
    clock_t utime, stime;
    int sleep_time;

    while (last > 0)
    {
        /* CPU Burst */
        times(&start_time);
        /* 其实只有t.tms_utime才是真正的CPU时间。但我们是在模拟一个
         * 只在用户状态运行的CPU大户，就像“for(;;);”。所以把t.tms_stime
         * 加上很合理。*/
        do
        {
            times(&current_time);
            utime = current_time.tms_utime - start_time.tms_utime;
            stime = current_time.tms_stime - start_time.tms_stime;
        } while (((utime + stime) / HZ) < cpu_time);
        last -= cpu_time;

        if (last <= 0)
            break;

        /* IO Burst */
        /* 用sleep(1)模拟1秒钟的I/O操作 */
        sleep_time = 0;
        while (sleep_time < io_time)
        {
            sleep(1);
            sleep_time++;
        }
        last -= sleep_time;
    }
}
```

该程序创建五个子进程，共三种类型，如下表所示：

| 进程类型                                   | 函数                    | 运行时间 | CPU占用时间 | IO占用时间 |
| ------------------------------------------ | ----------------------- | -------- | ----------- | ---------- |
| CPU密集型进程（一个进程占用10秒的CPU时间） | `cpuio_bound(10, 1, 0)` | 10s      | 10          | 0          |
| IO密集型进程（I/O时间是CPU时间的9倍）      | `cpuio_bound(10, 1, 9)` | 10s      | 1           | 9          |
| CPU和I/O各1秒钟轮回                        | `cpuio_bound(10, 1, 1)` | 10s      | 5           | 5          |

将`process.c`放入Linux0.11中编译并运行，测试结果如下：

![](.\images\Snipaste_2025-11-17_19-42-43.png)

### 3.2. log文件准备工作

操作系统启动后先要打开 `/var/process.log` ，然后在每个进程发生状态切换的时候向 `log` 文件内写入一条记录，以此实现把从操作系统启动到系统关机过程中所有进程的运行轨迹都记录在这一 `log` 文件中。

#### 3.2.1. 打开`log`文件

为了能尽早开始记录，应当在内核启动时就打开 `log` 文件。 内核的入口是 `init/main.c` 中的 `main()` ，为了使`linux 0.11`在启动之后就创建`process.log`并开始记录，需要修改内核入口代码`/init/main.c`。

按照实验提示增加如下代码：

```C
 setup((void *)&drive_info);                                         /* 加载文件系统 */
 (void)open("/dev/tty0", O_RDWR, 0);     /* 打开/dev/tty0，建立文件描述符0和/dev/tty0的关联 */
 (void)dup(0);                                                    /* 让文件描述符1也和/dev/tty0关联 */
 (void)dup(0);                                                    /* 让文件描述符2也和/dev/tty0关联 */
 (void)open("/var/process.log", O_CREAT | O_TRUNC | O_WRONLY, 0666); /* 打开/var/process.log文件 */
```

这段代码建立了文件描述符 `0` 、 `1` 和 `2` ，它们分别就是 `stdin` 、 `stdout` 和 `stderr` 。 这三者的值是系统标准，不可改变。 可以把 `log` 文件的描述符关联到 `3` ；文件系统初始化，描述符 `0` 、 `1` 和 `2` 关联之后，才能打开 `log` 文件，开始记录进程的运行轨迹。

> [!NOTE]
>
> 打开 `log` 文件的参数的含义是建立只写文件，如果文件已存在则清空已有内容。文件的权限是所有人可读可写。
>
> 每个参数含义：
>
> - `O_CREAT`: 如果文件不存在则创建
> - `O_TRUNC`: 如果文件存在则清空
> - `O_WRONLY`: 只写模式
> - `0666`: 文件权限 (`rw-rw-rw-`)

这样，文件描述符 `0` 、 `1` 、 `2` 和 `3` 就在进程 `1` 中建立了。根据 `fork()` 的原理，进程 `1` 之后的进程会继承这些文件描述符，所以就不必再 `open()` 它们。 

但实际上， `init()` 的后续代码和 `/bin/sh` 都会重新初始化它们；所以只有进程 `1` 的文件描述符肯定关联着 `log` 文件，这一点在接下来的写 `log` 中很重要。

#### 3.2.2. 写`log`文件

`log` 文件将被用来记录进程的状态转移轨迹。所有的状态转移都是在内核进行的。 在内核状态下， `write()` 功能失效。将实验提示中的 `fprintk()` 写入 `kernel/printk.c` 中。

`fprintk()` 的使用方式与 `C` 标准库函数 `fprintf()` 相似 ，唯一的区别是第一个参数是文件描述符，而不是文件指针。例如：

```C
fprintk(1, "The ID of running process is %ld", current->pid); //向stdout打印正在运行的进程的ID
fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'R', jiffies); //向log文件输出
```

### 3.3. 跟踪进程运行轨迹

该部分的重点在于寻找状态切换点，必须找到所有发生进程状态切换的代码点，并在这些点添加适当的代码，来输出进程状态变化的情况到 `log` 文件中。首先分析每种进程状态切换涉及的代码点。 

 Linux 0.11 中的进程状态有以下五种：

- 进程新建( `N` )
- 进入就绪态( `J` )
- 进入运行态( `R` )
- 进入阻塞态( `W` )
- 退出( `E` )

总的来说， `Linux 0.11` 支持四种进程状态的转移： 就绪到运行、运行到就绪、运行到睡眠和睡眠到就绪，此外还有新建和退出两种情况。 其中:

- 进程新建（`N` $\rightarrow$ `J`）: 在 `kernel/fork.c` 的 `copy_process` 函数中
- 就绪到运行（`J` $\rightarrow$  `R`）: 在 `kernel/sched.c` 的 `schedule` 函数中
- 运行到就绪（`R` $\rightarrow$  `J`）: 在 `kernel/sched.c` 的 `schedule` 函数中
- 运行到睡眠（`R` $\rightarrow$  `W`）: 在 `kernel/sched.c` 的 `sleep_on`、`interruptible_sleep_on`、`sys_pause`、 函数中
- 睡眠到就绪（`W`  $\rightarrow$ `R`）: 在 `kernel/sched.c` 的 `wake_up` 函数中
- 进程退出（`E`）: 在 `kernel/exit.c` 的 `do_exit` 、`sys_waitpid`函数中

#### 3.3.1. 进程新建（`N` $\rightarrow$ `J`）

`copy_process()`函数首先进行内存分配与PCB初始化，并通过`p->start_time = jiffies;` 设置开始时间后，进程进入就绪态 `N`；然后进行TSS配置，内存映射，资源管理，设置进程状态等操作，全部完成后进程进入运行态 `J`。

按照`/var/process.log` 文件的要求格式，添加输出进程状态的代码：

```C
int copy_process( ... ){
    ...
	p->start_time = jiffies;
    /* add */
    fprintk(3, "%ld\t%c\t%ld\n", last_pid, 'N', jiffies);
    /* finish */
    p->tss.back_link = 0;
	...
    p->state = TASK_RUNNING; /* do this last, just in case */
    /* add */
    fprintk(3, "%ld\t%c\t%ld\n", p->pid, 'J', jiffies);
    /* finish */
    return last_pid;
}
```

#### 3.3.2. 就绪到运行（`J` $\rightarrow$  `R`）和 运行到就绪（`R` $\rightarrow$  `J`）

调度函数 `schedule()`实现进程就绪态与运行态之间的切换，该函数首先遍历所有进程，检查是否有闹钟到期的进程，如果有，则向其发送 `SIGALRM` 信号；同时检查是否有进程处于可中断睡眠状态（`TASK_INTERRUPTIBLE`）且收到未被阻塞的信号，如果有，则将其状态设置为就绪态（`TASK_RUNNING`）；接着实现调度算法，选择下一个要运行的进程，该部分只是确定了下一个要运行的进程，此时还未进行进程状态的改变；最后使用`switch_to(next);`切换任务。

综上，需要在三处增加输出进程状态变化的代码：

- 若检查存在处于`TASK_INTERRUPTIBLE`的状态且收到未被阻塞的信号，则将这些进程的状态设置为就绪态`J`；
- 在实际任务切换之前，输出当前运行进程的状态转换为就绪态`J`；
- 在实际任务切换之前，输出下一个将要运行的进程的状态转换为运行态`R`。

`schedule()`函数修改后如下所示：

```C
void schedule(void)
{
    int i, next, c;
    struct task_struct **p;

    /* check alarm, wake up any interruptible tasks that have got a signal */

    for (p = &LAST_TASK; p > &FIRST_TASK; --p)
        if (*p)
        {
            if ((*p)->alarm && (*p)->alarm < jiffies)
            {
                (*p)->signal |= (1 << (SIGALRM - 1));
                (*p)->alarm = 0;
            }
            if (((*p)->signal & ~(_BLOCKABLE & (*p)->blocked)) &&
                (*p)->state == TASK_INTERRUPTIBLE){
                (*p)->state = TASK_RUNNING;
                /* add */
                fprintk(3, "%ld\t%c\t%ld\n", (*p)->pid, 'J', jiffies);
                /* finish */
            }
        }

    /* this is the scheduler proper: */

    while (1)
    {
        c = -1;
        next = 0;
        i = NR_TASKS;
        p = &task[NR_TASKS];
        while (--i)
        {
            if (!*--p)
                continue;
            if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
                c = (*p)->counter, next = i;
        }
        if (c)
            break;
        for (p = &LAST_TASK; p > &FIRST_TASK; --p)
            if (*p)
                (*p)->counter = ((*p)->counter >> 1) +
                                (*p)->priority;
    }
    /* add: 进程切换：如果下一个进程与当前进程不同，则将下一个进程设置为运行状态再切换*/
    if (task[next]->pid != current->pid)
    {
        if (current->state == TASK_RUNNING)
        {
            fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'J', jiffies);
        }
        fprintk(3, "%ld\t%c\t%ld\n", task[next]->pid, 'R', jiffies);
    }
    /* finish */
    switch_to(next);
}
```

#### 3.3.3.运行到睡眠（`R` $\rightarrow$  `W`）

##### （1）`sleep_on`函数使当前进程进入不可中断的睡眠状态，直到被其他进程显式地唤醒。

- 在设置 `current->state = TASK_UNINTERRUPTIBLE;` 后，当前进程由运行态转换至阻塞态，输出当前进程这一状态`W`；
- 在等待队列中下一个进程被唤醒后，由等待态转换至就绪态，输出进程这一状态`J`。

`sleep_on()`函数修改后如下：

```C
void sleep_on(struct task_struct **p)
{
    struct task_struct *tmp;

    if (!p)
        return;
    if (current == &(init_task.task))
        panic("task[0] trying to sleep");
    tmp = *p;
    *p = current;
    current->state = TASK_UNINTERRUPTIBLE;
    /* add */
    fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'W', jiffies); /*进程进入睡眠（堵塞）*/
    /* finish */
    schedule();
    if (tmp)
    {
        tmp->state = 0;
        /* add */
        fprintk(3, "%ld\t%c\t%ld\n", tmp->pid, 'J', jiffies);
        /* finish */
    }
}
```

##### （2）`interruptible_sleep_on`函数使当前进程进入可中断的睡眠状态，等待特定事件的发生被唤醒。

- 在设置 `current->state = TASK_INTERRUPTIBLE;` 后，当前进程由运行态转换至阻塞态，输出当前进程这一状态`W`；
- 在等待队列中有进程被唤醒后，由等待态转换至就绪态，输出进程这一状态`J`。

`interruptible_sleep_on`函数修改后如下：

```C
void interruptible_sleep_on(struct task_struct **p)
{
    struct task_struct *tmp;

    if (!p)
        return;
    if (current == &(init_task.task))
        panic("task[0] trying to sleep");
    tmp = *p;
    *p = current;
repeat:
    current->state = TASK_INTERRUPTIBLE;
    /* add */
    fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'W', jiffies);
    /* finish */
    schedule();
    if (*p && *p != current)
    {
        (**p).state = 0;
        /* add */
        fprintk(3, "%ld\t%c\t%ld\n", (**p).pid, 'J', jiffies);
        /* finish */
        goto repeat;
    }
    *p = NULL;
    if (tmp)
    {
        tmp->state = 0;
        /* add */
        fprintk(3, "%ld\t%c\t%ld\n", tmp->pid, 'J', jiffies);
        /* finish */
    }
}
```

##### （3）`sys_pause`在操作系统没有其他进程需要运行时执行，寻找可运行进程，实现 CPU 空闲时的循环等待。

若系统无任何额外的进程工作，则进程0会不断的调用`sys_pause()`阻塞自己，频繁在运行态和等待态之间切换，记录这些状态变化是无意义的，故发生这种情况时无需向`log`记录。

```C
int sys_pause(void)
{
    /* add: 若系统无任何额外的进程工作，则进程0会不断的调用sys_pause()阻塞自己，当这种情况发生时，无需向 log 记录*/
    if (current->state != TASK_INTERRUPTIBLE)
        fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'W', jiffies);
    /* finish */
    current->state = TASK_INTERRUPTIBLE;
    schedule();
    return 0;
}
```

#### 3.3.4. 睡眠到就绪（`W`  $\rightarrow$ `R`）

`wake_up`函数将等待队列中第一个进程唤醒，进程从等待态转移至就绪态，输出进程的新状态`J`。函数修改后如下：

```C
void wake_up(struct task_struct **p)
{
    if (p && *p)
    {
        (**p).state = 0;
        /* add */
        fprintk(3, "%ld\t%c\t%ld\n", (*p)->pid, 'J', jiffies);
        /* finish */
        *p = NULL;
    }
}
```

#### 3.3.5. 进程退出（`E`）

`kernel/exit.c`文件中有两个函数需要修改：

- `do_exit()`：处理进程的退出操作。释放进程占用的资源，更新进程状态，并通知父进程。在将进程状态设置为退出态之后，输出`E`。
- `sys_waitpid()`：等待子进程的状态变化。在将进程状态设置为可中断等待态之后，输出`W`。

```C
int do_exit(long code){
    ...
    current->state = TASK_ZOMBIE;
    /* add */
    fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'E', jiffies);
    /* finish */
    current->exit_code = code;
    ...
}
```

```C
int sys_waitpid(pid_t pid, unsigned long *stat_addr, int options){
    ...
    if (flag)
    {
        if (options & WNOHANG)
            return 0;
        current->state = TASK_INTERRUPTIBLE;
        /* add */
        fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'W', jiffies);
        /* finish */
        schedule();
        if (!(current->signal &= ~(1 << (SIGCHLD - 1))))
            goto repeat;
        else
            return -EINTR;
    }
    return -ECHILD;
}
```

### 3.4. 数据统计

上述代码修改工作全部完成后，重新编译Linux0.11内核，生成镜像文件`Image`，再启动`dbg-bochsgui246` 运行，在Linux0.11中运行编译好的`process`程序。

程序运行完毕后，执行`sync`命令，该命令会刷新 `cache` ，确保文件确实写入了磁盘。关闭`bochs`，使用挂载的方式将日志文件从 Linux 0.11的 `/var/process.log` 拷贝至宿主机 `~/oslab/files` 目录 下：

```shell
sudo ./mount-hdc
cp ./hdc/var/process.log ~/oslab/files
sudo umount hdc
```

打开`procss.log`，查看其中的内容，五中状态均有打印，如下图：

![](.\images\Snipaste_2025-11-18_11-34-37.png)

使用`tail -n 50 process.log > last.txt` 查看`process.log`最后50行的内容，如下图，可以看到也有其他进程建立和运行：

![](.\images\Snipaste_2025-11-18_11-37-51.png)

接下来，统计该程序建立的所有进程的等待时间、完成时间（周转时间）和运行时间，并计算平均等待时间，平均完成时间和吞吐量。运行 `./stat_log.py process.log 0 1 2 3 4 5 -g` （只统计 `PID` 为 `0` 、 `1` 、 `2` 、 `3` 、 `4` 和 `5` 的进程），得到结果如下图：

![](.\images\Snipaste_2025-11-18_11-40-28.png)

### 3.5. 修改时间片

创建新进程时，新进程会复制父进程的 counter, priority，后续进程运行时间片用完后，根据自身 counter, priority 重新计算。因此，修改进程 0 初始化的 priority 即可实现对时间片的修改。时间片相关参数在`linux-0.11/include/linux/sched.h`的宏`INIT_TASK`中定义，修改宏中的第三个值得到不同的时间片。

```C
/* 这里修改为10 */
#define INIT_TASK \
/* state etc */	{ 0,15,10, \
```

分别将时间片修改为5，10，25，40，并使用`stat_log.py`进行统计，结果依次如下：

![](.\images\Snipaste_2025-11-18_12-08-10.png)

![](E:\HitOSlab\lab5\images\Snipaste_2025-11-18_11-55-37.png)

![](.\images\Snipaste_2025-11-18_11-58-11.png)

![](.\images\Snipaste_2025-11-18_12-04-25.png)

## 4. 实验报告

- [x] `process.c` 
- [x] 日志文件建立成功
- [x] 能向日志文件输出信息
- [x] 5种状态都能输出
- [x] 调度算法修改

### 问题回答：

#### 4.1. 结合自己的体会，谈谈从程序设计者的角度看，单进程编程和多进程编程最大的区别是什么？

如果仅从计算机执行程序这个角度来看，那操作系统就相当于一个所有运行中程序的管理者，需要使用各种复杂的机制与策略来实现程序之间不会出现对系统共享资源的同时访问（互斥，保证状态迁移原子性），同时确保同步运行，保证程序每步状态迁移过程中的确定性；而多进程编程则强制需要编程者把自己作为像操作系统一样的程序的管理者，来人为的管理多个程序的这种互斥和同步关系，并协调它们完成一个任务目标。

从程序设计者的角度看，单进程与多进程编程的根本差别在于如何管理程序状态与手工处理并发：单进程直接共享状态、实现简单但容错差；多进程通过隔离提高可靠性和并行能力，但需要显式的通信与更高的工程成本。选择取决于性能需求、容错要求和设计者愿意承担的复杂性。下表给出了从可行的机制与工程设计角度，单进程编程和多进程编程的一些区别：

|                | 单进程编程                                                   | 多进程编程                                                   |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 内存与状态隔离 | 所有代码共享同一地址空间，数据访问直接且低开销               | 每个进程有独立地址空间，默认不共享内存（需要显式的共享内存机制）。这带来更强的故障隔离但也增加了数据同步代价 |
| 并发控制       | 按顺序执行，天然无并发问题。即使采用单进程多线程，仍是单地址空间的并发 | 能在多核上真正并行运行，且进程间独立调度，这在CPU 密集型任务通常更有效 |
| 通信与同步方式 | 单线程：函数调用<br />多线程：共享变量、锁等机制实现互斥     | 使用信号量、管道、消息队列、共享内存等，设计上更倾向“消息传递”而不是直接共享变量 |
| 容错性 / 隔离  | 单进程出错可能导致整个服务不可用                             | 多进程一个子进程崩溃通常不会直接导致其他进程崩溃，易于做重启策略 |
| 性能开销       | 不需要显示的进程切换，开销较小                               | 进程切换高于进程内函数/线程调用；但多进程能利用多核带来更好的吞吐 |
| 针对不同任务   | I/O 密集型任务采用单进程 + 异步/事件循环（或线程）通常更简单高效，且容易调试 | CPU密集型任务优先考虑多进程以充分利用多核；在有全局解释器锁（如` Python`的` CPython`）时尤其推荐多进程。 |



#### 4.2. 你是如何修改时间片的？仅针对样本程序建立的进程，在修改时间片前后， `log` 文件的统计结果（不包括Graphic）都是什么样？结合你的修改分析一下为什么会这样变化，或者为什么没变化？

##### 1. 修改时间片

修改`/include/linux/sched.h`文件里的`priority`宏定义实现时间片的修改，具体分析如下：

`Linux0.11` 的调度算法是选取 `counter` 值最大的就绪进程进行调度。 其中运行态进程（即 `current` ）的 `counter` 数值会随着时钟中断而不断减 `1` （时钟中断 `10ms` 一次）， 所以是一种比较典型的时间片轮转调度算法。另外，由上面的程序可以看出，当没有 `counter` 值大于 `0` 的就绪进程时， 要对所有的进程做

```
(*p)->counter = ((*p)->counter >> 1) + (*p)->priority
```

其效果是对所有的进程（包括阻塞态进程）都进行 `counter` 的衰减，并再累加 `priority` 值。 这样，对正被阻塞的进程来说，一个进程在阻塞队列中停留的时间越长，其优先级越大，被分配的时间片也就会越大。 所以总的来说， `Linux 0.11` 的进程调度是一种综合考虑进程优先级并能动态反馈调整时间片的轮转调度算法。

父进程`fork`产生新进程时，新进程会复制父进程的`counter`, `priority`，后续进程运行时间片用完后，根据自身 `counter`, `priority` 重新计算。因此修改`priority`即可实现对这些子进程时间片的修改。

##### 2. log文件的统计结果，在`3.5. 修改时间片`部分已展示，结果是通过 `stat_log.py` 进行统计得到的

##### 3. 分析

以下是实验过程中得到的每种情况下的平均周转时长和吞吐量统计：

| 时间片 | 平均周转时长 | 吞吐量 |
| ------ | ------------ | ------ |
| 5      | 1867.00      | 0.15   |
| 10     | 2875.00      | 0.10   |
| 15     | 4358.83      | 0.07   |
| 25     | 2056.17      | 0.13   |
| 40     | 2115.75      | 0.12   |

`Linux0.11`原本时间片为15，修改后平均周转时长和吞吐量均有升高；且以时间片为自变量，可以发现平均周转时长表现出先上升、后下降的趋势，吞吐量则先下降后上升。



> [!NOTE]
>
> 这个结果很奇怪，其他大多数人的结果似乎与我呈现相反的趋势，但我的操作流程似乎并没有问题，所以我以下的分析是个人对以上结果可能解释的推测。

对于CPU 密集型作业：增大时间片通常减少上下文切换开销，能提升吞吐量；但对短作业或等待 I/O 的作业，会因为长时间占用 CPU 导致等待时间增加，从而提高平均周转时间。而减小时间片时，CPU 一直工作在“切换开销”而不是“做计算”的环节中，所有进程完成时间还是都会变长，导致平均周转时间上升。

对于I/O 密集型或短任务为主的负载：理论上调大时间片对这种类型的任务并没有什么作用，但调大时间片会让 CPU 密集型任务霸占更久，导致整体 I/O 并发性下降，平均周转时间上升。而缩短时间片对这种类型的任务更有利，可以让短任务更快完成（看起来周转时间变小），但过短又会因上下文切换和缓存失效而降低有效吞吐量。不同混合比下，时间片调大/调小都会对不同指标产生相反影响。