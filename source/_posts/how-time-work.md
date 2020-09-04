title: Linux如何获取时间
tags: []
categories: []
date: 2020-08-25 09:42:00
---
## 背景

这几天和盛哥聊到了系统调用的开销，有些是io,有些是内存，突然提到了尽管像**获取时间**这种调用，有些对性能要求极高的系统也会做针对性优化。
连"获取时间“都要优化，nginx不就是这么处理么？为什么要这样处理?时间这个基础性的信息底层到底是怎样呢，而高级语言Java/C这些获取时间的方式又是怎么和底层交互的？这个问题激起了我极大的兴趣，希望能通过对“时间”的技术细节做层梳理和探索，加深对操作系统以及Java语言的理解。以生产环境服务器ubuntu为例，设定以下目标

- C/Java是如何获取当前时间
- Linux是怎么维护和衡量时间
- java获取当前时间的原理，是否存在性能问题
- nginx对获取时间做了什么优化

## C/Java是如何获取当前时间

从[The Linux Programming Interface
](https://man7.org/tlpi/) 上看，获取时间最常用的函数是** [gettimeofday]**，常见的中间件也是用该函数么？

```c
#include <sys/time.h>
int gettimeofday(struct timeval *tv, struct timezone *tz);
												Returns 0 on success, or –1 on error
```

### Redis 

扫了下redis的命令，发现redis有个[time](https://redis.io/commands/time) command. 调用该command能够返回当前服务器的时间戳，从[redis源码](https://github.com/redis/redis/blob/7bf665f125a4771db095c83a7ad6ed46692cd314/src/server.c#L3857)
)上看，redis也是调用了**gettimeofday**获取当前的时间

```c
void timeCommand(client *c) {
    struct timeval tv;

    /* gettimeofday() can only fail if &tv is a bad address so we
     * don't check for errors. */
    gettimeofday(&tv,NULL);
    addReplyArrayLen(c,2);
    addReplyBulkLongLong(c,tv.tv_sec);
    addReplyBulkLongLong(c,tv.tv_usec);
}
```

### Java

Java呢，是怎么获取时间的, System.currentTimeMillis()是java中获取当前时间的基础，java.lang.Date对象的构造也是通过调用System.currentTimeMillis()获取当前时间戳进行初始化。

[System源码](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/lang/System.java#l353)

```java
    /**
     * Returns the current time in milliseconds.  Note that
     * while the unit of time of the return value is a millisecond,
     * the granularity of the value depends on the underlying
     * operating system and may be larger.  For example, many
     * operating systems measure time in units of tens of
     * milliseconds.
     *
     * <p> See the description of the class <code>Date</code> for
     * a discussion of slight discrepancies that may arise between
     * "computer time" and coordinated universal time (UTC).
     *
     * @return  the difference, measured in milliseconds, between
     *          the current time and midnight, January 1, 1970 UTC.
     * @see     java.util.Date
     */
    public static native long currentTimeMillis();
```

好吧，该方法是一个native方法，那就只能继续往jvm源码上看,从jvm源码上看，java获取时间戳最终也是通过gettimeofday获取,殊途同归。

[jvm源码os_linux.cpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/os/linux/vm/os_linux.cpp#l1362)

```cpp
jlong os::javaTimeMillis() {
  timeval time;
  int status = gettimeofday(&time, NULL);
  assert(status != -1, "linux error");
  return jlong(time.tv_sec) * 1000  +  jlong(time.tv_usec / 1000);
}
```

好，继续往下一个问题

## Linux是怎么维护和衡量时间

总的来说，现代操作系统主板上会有个[Real Time Clock](https://en.wikipedia.org/wiki/Real-time_clock),记录着当前的时间，通过主板上的电池[CMOS Battery](https://en.wikipedia.org/wiki/Nonvolatile_BIOS_memory#CMOS_battery)维持,假如电池没电了,那我们启动后时间就会出现不正常的情况，通过api去获取时间当然的也会出现相应的问题问题。假如时间出现问题，可以通过手动或者自动的方式校准。  

服务器启动的时候就会读取当前时间，保存到kernel，记T1  
同时启动[TSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter),什么是TSC?简单来说就是一个64bit的寄存器，记录着服务器启动以来cpu的cycle,每个cycle的时间就是1/CPU频率，例如cpu 2GHZ, 那么就表示一个cycle的时间是0.5ns
所以获取当前时间的方式就是

```
当前时间 = T1 + TSC * 0.5  //时间的精度为1ns
```

除了TSC, Linux还有其他方式维护时间，但由于目前主流服务器上都是TSC,所以其他方式的不详述  
如何查看当前服务器支持的时间源

```
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
tsc hpet acpi_pm
```

如何查看当前服务器使用的时间源

```
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
tsc
```

Linux最常用的获取时间的接口是[gettimeofday], 但除了该方法，Linux还提供了[clock_gettime],该api提供了多种获取时间的方式，这里需要注意的是，[gettimeofday]最终也是调用了[clock_gettime]就是指定了clockid clock\_realtime的时间, 在新版本, linux提供多了一种clockid,[CLOCK\_REALTIME\_COARSE](https://lore.kernel.org/patchwork/patch/164350/),从manual上看该clockid提供了了一种新的选择“need very fast, but not fine-grained timestamps”。  

需要注意的，对于clockid CLOCK_MONOTONIC,这里获取的是一个单调递增的计数器，不受系统时间修改的影响，这个java也会使用到，先记着:)

|clockid|description|
|---|---|
|CLOCK_REALTIME|        System-wide  clock  that  measures  real (i.e., wall-clock) time.  Setting this clock requires appropriate privileges.  This clock is affected by discontinuous        jumps in the system time (e.g., if the system administrator manually changes the clock), and by the incremental adjustments performed by adjtime(3) and NTP.  |
|CLOCK_REALTIME_COARSE| (since Linux 2.6.32; Linux-specific)        A faster but less precise version of CLOCK_REALTIME.  Use when you need very fast, but not fine-grained timestamps.  |
|CLOCK_MONOTONIC|        Clock that cannot be set and represents monotonic time since some unspecified starting point.  This clock is not affected by discontinuous jumps in the  system        time (e.g., if the system administrator manually changes the clock), but is affected by the incremental adjustments performed by adjtime(3) and NTP.  |
|CLOCK_MONOTONIC_COARSE| (since Linux 2.6.32; Linux-specific)        A faster but less precise version of CLOCK_MONOTONIC.  Use when you need very fast, but not fine-grained timestamps.  |
|CLOCK_MONOTONIC_RAW| (since Linux 2.6.28; Linux-specific)        Similar to CLOCK_MONOTONIC, but provides access to a raw hardware-based time that is not subject to NTP adjustments or the incremental adjustments performed by        adjtime(3).  |
|CLOCK_BOOTTIME| (since Linux 2.6.39; Linux-specific)        Identical to CLOCK_MONOTONIC, except it also includes any time that the system is suspended.  This allows applications to get a suspend-aware  monotonic  clock        without having to deal with the complications of CLOCK_REALTIME, which may have discontinuities if the time is changed using settimeofday(2).  |
|CLOCK_PROCESS_CPUTIME_ID| (since Linux 2.6.12)        Per-process CPU-time clock (measures CPU time consumed by all threads in the process).  |
|CLOCK_THREAD_CPUTIME_ID| (since Linux 2.6.12)        Thread-specific CPU-time clock.|

接下来，我们就来对比各种clockid的性能。
可以参考[StackOverflow](https://stackoverflow.com/questions/6498972/faster-equivalent-of-gettimeofday/13096917#13096917)的这个回答，可以看出
CLOCK_REALTIME和CLOCK_REALTIME_COARSE的性能区别还是比较高，这样对于需要高性能但相对不精确的时间时还是可以选择CLOCK_REALTIME_COARSE的

>time (s) => 3 cycles  
ftime (ms) => 54 cycles  
gettimeofday (us) => 42 cycles  
clock_gettime (ns) => 9 cycles (CLOCK_MONOTONIC_COARSE)  
clock_gettime (ns) => 9 cycles (CLOCK_REALTIME_COARSE)  
clock_gettime (ns) => 42 cycles (CLOCK_MONOTONIC)  
clock_gettime (ns) => 42 cycles (CLOCK_REALTIME)  
clock_gettime (ns) => 173 cycles (CLOCK_MONOTONIC_RAW)  
clock_gettime (ns) => 179 cycles (CLOCK_BOOTTIME)  
clock_gettime (ns) => 349 cycles (CLOCK_THREAD_CPUTIME_ID)  
clock_gettime (ns) => 370 cycles (CLOCK_PROCESS_CPUTIME_ID)  
rdtsc (cycles) => 24 cycles  

好吧，上面只是别人的一些经验，直接在服务器上跑[测试](https://github.com/btorpey/clocks),可以看出CLOCK_REALTIME平均执行16.15ns, coarse版本的api执行的时间为0， 说明执行的时间小于 1/3.4 ns，好了，至此已经知道Linux常用获取时间的api的效率了

服务器CPU为 3.4GHz

|                  Method  |          min |          max |          avg |       median |        stdev|
|---|---|---|---|---|---|
|           CLOCK_REALTIME | 14.00        | 18.00        | 16.15        | 16.00        |  1.24|
|    CLOCK_REALTIME_COARSE |  0.00        |  0.00        |  0.00        |  0.00        |  0.00|
|          CLOCK_MONOTONIC | 14.00        | 19.00        | 15.88        | 16.50        |  1.23|
|      CLOCK_MONOTONIC_RAW | 63.00        | 67.00        | 64.87        | 65.00        |  1.23|
|   CLOCK_MONOTONIC_COARSE |  0.00        |  0.00        |  0.00        |  0.00        |  0.00|



## java获取当前时间的原理，是否存在性能问题

从上面已知， System.currentTimeMillis()最终调用是到了clock_realtime,该api执行时间大概16ns.
为什么java不用coarse版本的接口呢？从网上查了一轮资料，发现在2017年其实已经有人提了这个问题,具体问题可参考[Jdk Issue](http://localhost:4000/admin/#/posts/cke9ack1f000400vj079om36h)  
简单的说，官方回复是，该执行时间目前不是瓶颈，coarse版本对于旧版本的操作系统不一定支持，需要进行各种兼容判断，会带来额外的性能开销。

### java其他时间函数- System.nanoTime()
Java除了System.currentTimeMillis(),还有一个常用的获取时间api, System.nanoTime(). 这个方法容易望文生义，觉得就是返回当前时间戳的纳秒数。在服务器上执行以下代码

```java
  public static void main(String[] args) {
    System.out.println(System.currentTimeMillis());
    System.out.println(System.nanoTime());
  }
```

控制台打印

```
1598924672863
8616575473749327
```

这个输出有点奇怪，假如nanoTime输出的是是距离epoch至今的纳秒数，那么理论上值应该是 System.currentTimeMillis() * 10的9次方

从[JDK注释](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/lang/System.java#l356)上看， "can only be used to measure elapsed time", 注释说明了该api提供的是作用是度量elapsed time, 而非某个具体的时间，该返回的时间也不受系统时间（wall clock）修改影响，例如ntp的时间矫正或者人工调整时间。

```java
 /**
     * Returns the current value of the running Java Virtual Machine's
     * high-resolution time source, in nanoseconds.
     *
     * <p>This method can only be used to measure elapsed time and is
     * not related to any other notion of system or wall-clock time.
     * The value returned represents nanoseconds since some fixed but
     * arbitrary <i>origin</i> time (perhaps in the future, so values
     * may be negative).  The same origin is used by all invocations of
     * this method in an instance of a Java virtual machine; other
     * virtual machine instances are likely to use a different origin.
     
     * <p> For example, to measure how long some code takes to execute:
     *  <pre> {@code
     * long startTime = System.nanoTime();
     * // ... the code being measured ...
     * long estimatedTime = System.nanoTime() - startTime;}</pre>
     *
```

那么nanoTime有什么应用场景呢，其实在jdk里面有些类也特意使用了nanoTime.  
Java有个表经典的问题，**为什么使用ScheduledThreadPoolExecutor而非Timer**.   
[Java Concurrency in Practice
](http://jcip.net/)里对比[Timer](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/util/Timer.java#l180)和ScheduledThreadPoolExecutor提到以下

>Timer does have support for scheduling based on absolute, not relative time, so that tasks can be sensitive to changes in the system clock; ScheduledThreadPoolExecutor supports only relative time.

这个的意思是对于Timer的定时任务，会受到系统时间的影响，为什么呢？
因为[Timer](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/util/Timer.java#l180)计算执行时间是通过System.currentTimeMillis(), 该时间是绝对时间，修改了系统时间，该时间相应就产生了变化

```java
public void schedule(TimerTask task, long delay) {
 if (delay < 0)
     throw new IllegalArgumentException("Negative delay.");
 sched(task, System.currentTimeMillis()+delay, 0);
}
```

而[ScheduledThreadPoolExecutor](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/util/concurrent/ScheduledThreadPoolExecutor.java#l496)是通过nanoTime获取时间

```java
long triggerTime(long delay) {
       return now() +
              ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
}
final long now() {
       return System.nanoTime();
}
```

那么nanoTime这个api获取的是什么时间？从jvm源码上看，对于支持supports_monotonic_clock该时间源的，获取的就是CLOCK_MONOTONIC这个时间，这个时间是单调递增， 不受系统时间影响，所以ScheduledThreadPoolExecutor使用该clock source会更加的合理。

[os_linux.cpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/os/linux/vm/os_linux.cpp#l1453)

```cpp
jlong os::javaTimeNanos() {
  if (Linux::supports_monotonic_clock()) {
    struct timespec tp;
    int status = Linux::clock_gettime(CLOCK_MONOTONIC, &tp);
    assert(status == 0, "gettime error");
    jlong result = jlong(tp.tv_sec) * (1000 * 1000 * 1000) + jlong(tp.tv_nsec);
    return result;
  } else {
    timeval time;
    int status = gettimeofday(&time, NULL);
    assert(status != -1, "linux error");
    jlong usecs = jlong(time.tv_sec) * (1000 * 1000) + jlong(time.tv_usec);
    return 1000 * usecs;
  }
}
```

## nginx是如何做优化

简单说， nginx是通过时间缓存优化时间获取的效率，定时更新缓存，获取时间时从该缓存获取

待补充

## 引用
1. [how-do-computers-keep-track-of-time](https://cs.stackexchange.com/questions/54933/how-do-computers-keep-track-of-time/54935#54935?newreg=a3c82ae7da40419fac87f267f2f76623)  
1. [select posix clocks](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/7/html/reference_guide/sect-posix_clocks#CLOCK_MONOTONIC_COARSE_and_CLOCK_REALTIME_COARSE)
1. [fast equivalent of gettimeofday](https://stackoverflow.com/questions/6498972/faster-equivalent-of-gettimeofday)
1. [HPET vs TSC](https://www.chromium.org/chromium-os/how-tos-and-troubleshooting/tsc-resynchronization)

[gettimeofday]:https://man7.org/linux/man-pages/man2/gettimeofday.2.html
[clock_gettime]:https://www.man7.org/linux/man-pages/man2/clock_gettime.2.html