￼ ￼ ￼ ￼
# Linux Performance Analysis in 60,000 Milliseconds

You login to a Linux server with a performance issue:
what do you check in the first minute?

At Netflix we have a massive EC2 Linux cloud, and numerous performance analysis tools to monitor and investigate its performance.

These include Atlas for cloud­wide monitoring, and Vector for on­demand instance analysis.

While those tools help us solve most issues, we sometimes need to login to an instance and run some standard Linux performance tools.

In this post, the Netflix Performance Engineering team will show you the first 60 seconds of an optimized performance investigation at the command line, using standard Linux tools you should have available.

## First 60 Seconds: Summary
In 60 seconds you can get a high level idea of system resource usage and running processes by running the following ten commands.

Look for errors and saturation metrics, as they are both easy to interpret, and then resource utilization.

Saturation is where a resource has more load than it can handle, and can be exposed either as the length of a request queue, or time spent waiting.

```
uptime  
dmesg | tail vmstat 1  
mpstat -P ALL 1 pidstat 1  
iostat -xz 1 free -m  
sar -n DEV 1  
sar -n TCP,ETCP 1 top
```

Some of these commands require the sysstat package installed.
The metrics these commands expose will help you complete some of the USE Method: a methodology for locating performance bottlenecks.

This involves checking utilization, saturation, and error metrics for all resources (CPUs, memory, disks, e.t.c.).

Also pay attention to when you have checked and exonerated a resource, as by process of elimination this narrows the targets to study, and directs any follow on investigation.

The following sections summarize these commands, with examples from a production system. For more information about these tools, see their man pages.

#### 1. uptime

```
$ uptime  
23:51:26up21:31, 1user, loadaverage:30.02,26.43,19.02

```
This is a quick way to view the load averages, which indicate the number of tasks (processes) wanting to run. On Linux systems, these numbers include processes wanting to run on CPU, as well as processes blocked in uninterruptible I/O (usually disk I/O).

This gives a high level idea of resource load (or demand), but can’t be properly understood without other tools.

Worth a quick look only.

The three numbers are exponentially damped moving sum averages with a 1 minute, 5 minute, and 15 minute constant.

The three numbers give us some idea of how load is changing over time.

For example, if you’ve been asked to check a problem server, and the 1 minute value is much lower than the 15 minute value, then you might have logged in too late and missed the issue.

In the example above, the load averages show a recent increase, hitting 30 for the 1 minute value, compared to 19 for the 15 minute value.

That the numbers are this large means a lot of something: probably CPU demand; vmstat or mpstat will confirm, which are commands 3 and 4 in this sequence.

#### 2. dmesg | tail

```
$ dmesg | tail  
######   此处暂时省略   #####\#
```

This views the last 10 system messages, if there are any.

Look for errors that can cause performance issues.

The example above includes the oom­killer, and TCP dropping a request.
Don’t miss this step! dmesg is always worth checking.



#### 3. vmstat 1

```
$ vmstat 1  
######   此处暂时省略   #####\#
```

Short for virtual memory stat, vmstat(8) is a commonly available tool (first created for BSD decades ago).

 It prints a summary of key server statistics on each line.

vmstat was run with an argument of 1, to print one second summaries.

The first line of output (in this version of vmstat) has some columns that show the average since boot, instead of the previous second.

For now, skip the first line, unless you want to learn and remember which column is which. Columns to check:

r:
Number of processes running on CPU and waiting for a turn.
This provides a better signal than load averages for determining CPU saturation, as it does not include I/O.

To interpret: an “r” value greater than the CPU count is saturation.

free: Free memory in kilobytes.

If there are too many digits to count, you have enough free memory.

The “free ­m” command, included as command 7, better explains the state of free memory.

si, so:
Swap­ins and swap­outs. If these are non­zero, you’re out of memory.

us, sy, id, wa, st:
These are breakdowns of CPU time, on average across all CPUs.
They are user time, system time (kernel), idle, wait I/O,
and stolen time (by other guests, or with Xen, the guest's own isolated driver domain).
The CPU time breakdowns will confirm if the CPUs are busy, by adding user + system time.
A constant degree of wait I/O points to a disk bottleneck;
this is where the CPUs are idle, because tasks are blocked waiting for pending disk I/O. You can treat wait I/O as another form of CPU idle, one that gives a clue as to why they are idle.

System time is necessary for I/O processing.

 A high system time average, over 20%, can be interesting to explore further: perhaps the kernel is processing the I/O inefficiently.

In the above example, CPU time is almost entirely in user­level, pointing to application level usage instead.

 The CPUs are also well over 90% utilized on average.
This isn’t necessarily a problem; check for the degree of saturation using the “r” column.

#### 4. mpstat ­P ALL 1

```
$ mpstat -P ALL 1
#######  此处暂时省略
```

This command prints CPU time breakdowns per CPU, which can be used to check for an imbalance.

A single hot CPU can be evidence of a single­threaded application.

#### 5. pidstat 1

```
$ pidstat 1
```

Pidstat is a little like top’s per­process summary, but prints a rolling summary instead of clearing the screen.

This can be useful for watching patterns over time, and also recording what you saw (copy­n­paste) into a record of your investigation.

The above example identifies two java processes as responsible for consuming CPU.

The %CPU column is the total across all CPUs; 1591% shows that that java processes is consuming almost 16 CPUs.


#### 6. iostat ­xz 1

```
$ iostat -xz 1   ######省略
```

This is a great tool for understanding block devices (disks), both the workload applied and the resulting performance.
Look for:
r/s, w/s, rkB/s, wkB/s:
These are the delivered reads, writes, read Kbytes, and write Kbytes per second to the device.
Use these for workload characterization.
A performance problem may simply be due to an excessive load applied.

await:
The average time for the I/O in milliseconds.
This is the time that the application suffers, as it includes both time queued and time being serviced. Larger than expected average times can be an indicator of device saturation, or device problems.

avgqu­sz:
The average number of requests issued to the device. Values greater than 1 can be evidence of saturation (although devices can typically operate on requests in parallel, especially virtual devices which front multiple back­end disks.)

%util:
Device utilization.
This is really a busy percent, showing the time each second that the device was doing work. Values greater than 60% typically lead to poor performance (which should be seen in await), although it depends on the device.
Values close to 100% usually indicate saturation.

If the storage device is a logical disk device fronting many back­end disks, then 100% utilization may just mean that some I/O is being processed 100% of the time, however, the back­end disks may be far from saturated, and may be able to handle much more work.
￼ ￼ ￼ ￼
Bear in mind that poor performing disk I/O isn’t necessarily an application issue.

Many techniques are  typically used to perform I/O asynchronously, so that the application doesn’t block and suffer the latency directly (e.g., read­ahead for reads, and buffering for writes).

#### 7. free ­m

```
$ free -m
total used free shared buffers cached
Mem: 245998 24545 221453 83 -/+ buffers/cache: 23944 222053  
Swap: 0 0 0
```

The right two columns show:  
buffers: For the buffer cache, used for block device I/O.
cached: For the page cache, used by file systems.
￼ ￼
We just want to check that these aren’t near­zero in size, which can lead to higher disk I/O (confirm using iostat), and worse performance. The above example looks fine, with many Mbytes in each.

The “­/+ buffers/cache” provides less confusing values for used and free memory. Linux uses free memory for the caches, but can reclaim it quickly if applications need it. So in a way the cached memory should be included in the free memory column, which this line does. There’s even a website, linuxatemyram, about this confusion.

It can be additionally confusing if ZFS on Linux is used, as we do for some services, as ZFS has its own file system cache that isn’t reflected properly by the free ­m columns. It can appear that the system is low on free memory, when that memory is in fact available for use from the ZFS cache as needed.

#### 8. sar ­n DEV 1

```
$ sar -n DEV 1

```

Use this tool to check network interface throughput:
rxkB/s and txkB/s, as a measure of workload, and also to check if any limit has been reached.

In the above example, eth0 receive is reaching 22 Mbytes/s, which is 176 Mbits/sec (well under, say, a 1 Gbit/sec limit).

This version also has %ifutil for device utilization (max of both directions for full duplex), which is something we also use Brendan’s nicstat tool to measure. And like with nicstat, this is hard to get right, and seems to not be working in this example (0.00).

#### 9. sar ­n TCP,ETCP 1
```
`$ sar -n TCP,ETCP 1
```
`
active/s: Number of locally­initiated TCP connections per second (e.g., via connect()).

passive/s: Number of remotely­initiated TCP connections per second (e.g., via accept()).

retrans/s: Number of TCP retransmits per second.

The active and passive counts are often useful as a rough measure of server load: number of new accepted connections (passive), and number of downstream connections (active).

It might help to think of active as outbound, and passive as inbound, but this isn’t strictly true (e.g., consider a localhost to localhost connection).

Retransmits are a sign of a network or server issue; it may be an unreliable network (e.g., the public Internet), or it may be due a server being overloaded and dropping packets. The example above shows just one new TCP connection per­second.

#### 10. top
```
`$ top  
```
`
The top command includes many of the metrics we checked earlier.
It can be handy to run it to see if anything looks wildly different from the earlier commands, which would indicate that load is variable.

A downside to top is that it is harder to see patterns over time, which may be more clear in tools like vmstat and pidstat, which provide rolling output.

Evidence of intermittent issues can also be lost if you don’t pause the output quick enough (Ctrl­S to pause, Ctrl­Q to continue), and the screen clears.

## Follow­on Analysis

There are many more commands and methodologies you can apply to drill deeper.
See Brendan’s Linux Performance Tools tutorial from Velocity 2015, which works through over 40 commands, covering observability, benchmarking, tuning, static performance tuning, profiling, and tracing.

Tackling system reliability and performance problems at web scale is one of our passions.

If you would like to join us in tackling these kinds of challenges we are hiring!
Posted by Brendan Gregg
