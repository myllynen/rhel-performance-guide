# RHEL Performance Guide

## Introduction

This page provides a compact and to-the-point [Red Hat Enterprise Linux
(RHEL)](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux)
CPU, memory, IO, and network performance guide covering most relevant
tuning parameters and monitoring commands and tools. Diagnosing and
monitoring individual processes is also covered.

Topics like designing cloud-native applications, optimizing containers,
and tuning virtualization platforms are out of scope for this document.
Specialized workloads such as HPC, NFV/VNF, or RT (high-performance
computing, network function virtualization / virtualized network
functions, real-time) are not the focus for this document but some
aspects discussed here would be beneficial with those workloads as well.

The decision when to spend hours and days with performance tuning and
monitoring instead of merely throwing more (virtual) hardware to the
problem is left to the reader. Depending on the case either approach
might turn out to be the more cost-effective one. However, some of the
considerations and hints below might provide insight when and what kind
of additional resources would be most beneficial in a particular
situation. In general, right-sizing (virtual) hardware might be
difficult without good understanding of the workload characteristics.

Web based performance monitoring tools like [basic Cockpit
setup](https://opensource.com/article/20/11/cockpit-server-management),
system performance visualization with [PCP](https://pcp.io/) (see [part
1](https://www.redhat.com/en/blog/visualizing-system-performance-rhel-8-using-performance-co-pilot-pcp-and-grafana-part-1),
[part
2](https://www.redhat.com/en/blog/visualizing-system-performance-rhel-8-using-performance-co-pilot-pcp-and-grafana-part-2),
and [part
3](https://www.redhat.com/en/blog/visualizing-system-performance-rhel-8-part-3-kernel-metric-graphing-performance-co-pilot-grafana-and-bpftrace)),
and [Prometheus](https://prometheus.io/) are often extremely helpful and
should be considered especially in larger environments.

The reader is expected to be familiar with basic operating system
concepts and terms and understand the output of the example commands.
Although exact understanding of technologies like
[eBPF](https://www.brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html)
is not mandatory it might be helpful in many cases.

For a complete book on system performance see
https://www.brendangregg.com/systems-performance-2nd-edition-book.html.

For troubleshooting tips see
https://github.com/myllynen/rhel-troubleshooting-guide.

For illustration in which areas different tools operate see
https://www.brendangregg.com/Perf/linux_observability_tools.png.

For a PCP primer see
[Introduction to storage performance analysis with PCP](https://access.redhat.com/articles/2450251).

## Process Monitoring

Process monitoring related documentation references:

* [RHEL Getting started with perf guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/getting-started-with-perf_monitoring-and-managing-system-status-and-performance)
* [RHEL Getting started with flamegraphs guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/getting-started-with-flamegraphs_monitoring-and-managing-system-status-and-performance)
* [Linux perf-top basics: understand the % blog post](https://blog.dbi-services.com/linux-perf-top-basics-understand-the/)
* [Linux Performance page by Brendan Gregg](https://www.brendangregg.com/linuxperf.html)
* [perf Examples page by Brendan Gregg](https://www.brendangregg.com/perf.html)
* [bpftrace GitHub page](https://github.com/iovisor/bpftrace)
* [bcc GitHub page](https://github.com/iovisor/bcc)

### Overall System Status

Basic commands to see the current status of the system:

```
top
pcp atop
pcp htop
vmstat -S m -w 2
systemd-cgtop -d 2
# Report vmstat -w like statistics using MBs
pmrep -b MB -t 2 :vmstat-w
# Report system overall process state statistics
pmrep -g -p -t 2 :proc-os-stats
```

### Basic Process Information

When monitoring the system and processes the overhead caused by
different approaches and tools must be kept in mind especially in
production environments.

For a similar task
[perf(1)](https://man7.org/linux/man-pages/man1/perf.1.html) usually
introduces less overhead than
[strace(1)](https://man7.org/linux/man-pages/man1/strace.1.html) which
in turn might cause less overhead than
[ltrace(1)](https://man7.org/linux/man-pages/man1/ltrace.1.html). A
targeted [BPF](https://www.brendangregg.com/ebpf.html) tool most often
causes less overhead than
[tcpdump(8)](https://man7.org/linux/man-pages/man8/tcpdump.8.html). Any
tool will introduce some overhead, perhaps using
[cat(1)](https://man7.org/linux/man-pages/man1/cat.1.html) to print
values from the
[proc(5)](https://man7.org/linux/man-pages/man5/proc.5.html) files being
the tool causing least overhead but often also being the last
user-friendly option.

```
# Process execution time, basic memory and IO statistics
/usr/bin/time -v COMMAND
# Snapshot view of processes
pstree -Aalp
# Live view of processes
# Use -p PID1 -p PID2 to monitor pid-1 and pid-2 only,
# use 'f' to enter field management to enable field like
# number of threads, last used CPU, swapped size, wchan,
# use 'E'/'e' to change scale, 'u' to filter by user
top
# Alternative way for live view of selected processes
# Use comm instead of args to see command names only,
# use -T and tid to display all threads of processes
watch -d=cumu -n 2 \
  "ps -q PID1,PID2 axwwo user,pid,ppid,psr,%cpu,%mem,vsz,rss,wchan:21,stat,start,cputime,args"
# Details of selected processes
pidstat -h -d -r -u -w -C '.*PROCNAME.*' 2
# Alternative way to monitor process details
pmrep -gp -t 2 -i '.*PROCNAME.*' :proc-info :proc-essential
```

### Process CPU Usage Related Information

```
# Report 5 most CPU using processes
pmrep -1gU -t 2 -J 5 proc.hog.cpu
# Process CPU/scheduling statistics
pmrep -gp -t 2 -i '.*PROCNAME.*' :proc-cpu :proc-cpu-ext
# Report process CPU usage details
perf top -Kgnv -d 2 -e cycles:u -s cpu,pid,comm,dso,sym -p PID
```

### Process Memory Usage Related Information

```
# Report 5 most memory using processes
pmrep -1gU -t 2 -J 5 proc.hog.mem
# Process memory usage statistics
pmrep -gp -t 2 -i '.*PROCNAME.*' :proc-mem :proc-mem-ext
# Show process per-NUMA-node memory statistics
watch -d=cumu -n 2 numastat -p PID
# Summary of memory related system calls
strace -fc -e trace=%memory COMMAND
```

### Process IO Usage Related Information

```
# Show IO activity for a process
iotop -k -d 2 -p PID
# Report 5 most IO using processes
pmrep -1gU -t 2 -J 5 proc.hog.disk
# Process IO statistics
pmrep -gp -t 2 -i '.*PROCNAME.*' :proc-io :proc-io-ext
# Trace file operations related system calls
strace -CfttTy -e trace=%file,%desc -p PID
```

### Process Network Usage Related Information

```
# Report 5 most network using processes
# Requires PCP BCC PMDA netproc module installed (pcp-5.2.3)
pmrep -1gU -t 2 -J 5 proc.hog.net
# Process network usage statistics
# Requires PCP BCC PMDA netproc module installed (pcp-5.2.3)
# Requires default pmrep.conf update for netproc (pcp-5.3.0)
pmrep -gp -t 2 -i '.*PROCNAME.*' :proc-net :proc-net-ext
# Trace network related system calls
strace -CfttTy -e trace=%network -p PID
```

### Process Overview, Limits, Details, and Tracing

```
# Show various summaries and details for a process
pmrep -p -t 2 -i '.*PROCNAME.*' :proc<TAB>
```

```
# Show IPC information
ipcs -p | grep PID
ipcs -a | less
# Show process limits
cat /proc/PID/limits
# Show process systemd unit and service limits
systemctl status PID
systemctl show SERVICE | grep ^Limit
```

```
# List locks of a process
lslocks | grep PROCNAME
# List open files of a process,
# use grep -v instead for network and IPC sockets
lsof -VanP -p PID | grep -e REG -e DIR
```

```
# Show counts of library calls and times,
# use -S to include system calls as well
ltrace -c COMMAND
# Trace locking related library calls
ltrace -CfttT -e "*lock*" COMMAND
# Trace library calls of a running process
ltrace -CfttT -p PID
```

```
# Show counts of system calls and times
strace -fc -S calls COMMAND
perf trace -s COMMAND
# Show failed system calls only
strace -fZ COMMAND
perf trace --failure --errno-summary -S COMMAND
# Trace system calls of a running process
strace -CfttTy -p PID
perf trace -S -p PID
# Trace set of system calls only
strace -CfttTy -e trace=%ipc COMMAND
perf trace -e 'syscalls:sys_enter_sem*' \
  -e 'syscalls:sys_enter_msg*' -e 'syscalls:sys_enter_shm*' COMMAND
```

```
# Report page migrations
perf stat -e migrate:mm_migrate_pages -I 2000 -p PID
# Report page faults
perf stat -e page-faults -I 2000 -p PID
```

```
# Record process activity for 30 seconds,
# add -s for per-thread recording
perf record -g -p PID -- sleep 30
# Report process activity,
# add -T for per-thread reporting,
# add --show-cpu-utilization for CPU utilization,
# add --stdio for text based reporting
perf report -nv -s cpu,pid,comm,dso
```

```
# Report process user-space CPU cycle and cache stats
perf stat -e cycles:u -d -I 2000 -p PID
# Report process scheduler event stats
perf stat -e 'sched:*' -I 2000 -p PID 2>&1 \
  | grep -v " 0 " | grep -v "not counted"
# Report process system calls stats
perf stat -e 'syscalls:sys_enter_*' -I 2000 -p PID 2>&1 \
  | grep -v " 0 " | grep -v "not counted"
```


```
# Record process scheduling details to see why not idle
perf sched record -g COMMAND
# Show process scheduling state changes
perf sched timehist -Mw --state | grep PROCNAME
```

## CPU Tuning and Monitoring

CPU related documentation references:

* [RHEL Reviewing a system using tuna interface guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/reviewing-a-system-using-tuna-interface_monitoring-and-managing-system-status-and-performance)
* [RHEL Monitoring system performance with perf guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/monitoring-system-performance-with-perf_monitoring-and-managing-system-status-and-performance)
* [RHEL Configuring an operating system to optimize CPU utilization guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/configuring-an-operating-system-to-optimize-cpu-utilization_monitoring-and-managing-system-status-and-performance)
* [RHEL RT Tuning Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/8/html/tuning_guide/index)
* [RHEL Managing, monitoring, and updating the kernel guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/index)
* [RHEL Performance Analysis and Tuning PDF](https://www.redhat.com/cms/managed-files/Handout%20Performance%20Analysis%20and%20Tuning%20Red%20Hat%20Enterprise%20Linux%202019.pdf)
* [CPU Utilization is Wrong blog post](https://www.brendangregg.com/blog/2017-05-09/cpu-utilization-is-wrong.html)
* [Linux tracing systems & how they fit together blog post](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)
* [Linux Performance page by Brendan Gregg](https://www.brendangregg.com/linuxperf.html)

### Hardware CPU Configuration

There are usually a few CPU related hardware configurations available.
On performance critical systems all power saving options (including C-
and P-states) should be disabled and possible workload profile setting
should be set to maximum performance. Simultaneous multithreading (SMT)
known as HyperThreading on Intel processors should be disabled for most
latency-sensitive workloads. Turbo Boost should be enabled if available.

```
# Show current CPU setup details
lscpu
# Show current CPU status summary
lscpu -e
# Show compact NUMA layout
numactl -H
```

### System-wide Services

[irqbalance(1)](https://www.mankier.com/1/irqbalance) can be run as a
service that distributes hardware interrupts evenly across cores to
improve system performance. Except for specific cases (like certain HPC,
NFV, or RT workloads where manual IRQ affinity setup might be needed)
irqbalance service should be enabled on all systems.

[tuned(8)](https://www.mankier.com/8/tuned) is a service that configures
system performance parameters according to the selected performance
profile. Examples of [profiles provided by
tuned](https://www.mankier.com/7/tuned-profiles) include _desktop_,
_virtual-host_, _virtual-guest_, _latency-performance_, and
_throughput-performance_. tuned service with suitable profile should be
enabled on all systems. Further performance tuning can be considered if
a default tuned profile does not provide optimal settings for a
particular workload.

```
# Show currently active tuned profiles
tuned-adm active
# List available tuned profiles
tuned-adm list
# Apply tuned profile virtual-guest
tuned-adm profile virtual-guest
```

tuned profiles change system parameters related to CPU scheduling,
memory, IO, and networking. It can be helpful to investigate what
parameters different tuned profiles are altering by reviewing the tuned
profile configuration files under _/usr/lib/tuned_.

Custom tuned profiles can be created if needed, see
[RHEL Customizing Tuned profiles guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/customizing-tuned-profiles_monitoring-and-managing-system-status-and-performance).

### System-wide Configuration and CPU Related Kernel Parameters

Many of the tuned profiles set these up properly for most cases.

For larger applications refer to vendor documentation for exact
recommendations and consider application parameter tuning as well.

<pre>
# RHEL 8 boot parameter to provide PSI metrics under /proc/pressure
# <a href="https://access.redhat.com/solutions/5211481">Is Pressure Stall Information (PSI) supported in RHEL?</a>
# <a href="https://facebookmicrosites.github.io/psi/docs/overview.html">Getting Started with PSI</a>
psi=1
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/769683">Why is the scaling_governor file missing on the RHEL system?</a>
echo performance > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
</pre>

```
# Show processor power related values set by tuned profiles
# cpupower is provided by the kernel-tools package
cpupower frequency-info
cpupower idle-info
```

```
# Show processor performance bias value set by tuned profiles
# x86_energy_perf_policy is provided by the kernel-tools package
x86_energy_perf_policy -r
```

<pre>
# Kernel scheduler related sysctl parameters set by tuned profiles
# <a href="https://access.redhat.com/solutions/177953">About use of kernel parameter 'sched_*'</a>
kernel.sched_wakeup_granularity_ns = N
kernel.sched_min_granularity_ns = N
kernel.sched_migration_cost_ns = N
kernel.sched_autogroup_enabled = 1
</pre>

<pre>
# Latency-sensitive workloads might benefit from
# system partitioning and dynamic tickless behavior
# <a href="https://access.redhat.com/articles/3720611">Discussion of isolcpus=, nohz_full= and rcu_nocbs= kernel parameters</a>
# <a href="https://access.redhat.com/solutions/2273531">All about nohz_full kernel parameter</a>
nohz_full=2-5,8-11 isolcpus=2-5,8-11
</pre>

```
# Verify current isolated CPUs and nohz_full configuration
cat /sys/devices/system/cpu/isolated
cat /sys/devices/system/cpu/nohz_full
```

<pre>
# Other parameters to consider for highly latency-sensitive workloads
# <a href="https://access.redhat.com/solutions/367773">How do I disable MCE function?</a>
mce=ignore_ce
skew_tick=1
</pre>

Further tuning tips especially for low latency and real-time workloads
are described in the RHEL RT tuning guide and RHKB articles:

* [RHEL RT Tuning Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/8/html/tuning_guide/index)
* [What are CPU "C-states" and how to disable them if needed?](https://access.redhat.com/solutions/202743)
* [Power management features and latency spikes](https://access.redhat.com/articles/65410)

### Additional Configurations and Tools

For a system partitioning example see
[RHEL Performance Analysis and Tuning PDF](https://www.redhat.com/cms/managed-files/Handout%20Performance%20Analysis%20and%20Tuning%20Red%20Hat%20Enterprise%20Linux%202019.pdf).

[tuna(8)](https://www.mankier.com/8/tuna) tool can help reducing
complexity of system tuning tasks:

* [RHEL Reviewing a system using tuna interface guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/reviewing-a-system-using-tuna-interface_monitoring-and-managing-system-status-and-performance)
* [How can I reduce jitter by using CPU and IRQ pinning with tuna?](https://access.redhat.com/solutions/2171211)
* [How can I reduce jitter by using CPU and IRQ pinning without using tuna?](https://access.redhat.com/solutions/2144921)

[nice(1)](https://man7.org/linux/man-pages/man1/nice.1.html) can be used
to change and run a process with modified scheduling priority.
[chrt(1)](https://man7.org/linux/man-pages/man1/chrt.1.html) can be used
to check and set real-time scheduling attributes of a process.
[taskset(1)](https://man7.org/linux/man-pages/man1/taskset.1.html) can
be used to check and set CPU affinity of a process.
[numactl(8)](https://man7.org/linux/man-pages/man8/numactl.8.html) can
be used to run processes with a specific NUMA binding policy.

```
# Pin process 1234 to CPU core 2
taskset -cp 2 1234
```

```
# Move sshd to CPU cores 2-3, httpd to CPU cores 4-7
tuna -c 2-3 -t sshd -m -c 4-7 -t httpd -m
```

### CPU Related Monitoring and Testing

#### System-wide CPU Monitoring

```
# Show CPU and scheduler related statistics
top
sar -q 2
sar -u ALL 2
sar -P ALL 2
vmstat -S m -w 2
systemd-cgtop -d 2
mpstat -u -I SUM -P ALL 2
# Report kernel PSI CPU metrics
pmrep -gp -t 2 kernel.all.pressure.cpu
# Report various CPU related metrics
pmrep kernel<TAB>
```

```
# Show instructions-per-cycle (IPC) statistics,
# higher isns-per-cycle hint CPU bound workload
perf stat -a -- sleep 10
# Report CPU migrations
perf stat -e migrations -I 2000 -a
# Record system-wide activity for 30 seconds
perf record -g -a -- sleep 30
# Report activity,
# add -T for per-thread reporting,
# add --show-cpu-utilization for CPU utilization,
# add --stdio for text based reporting
perf report -n -s cpu,pid,comm,dso
```

See also [PCP](https://pcp.io/) tools
[pcp-atop(1)](https://man7.org/linux/man-pages/man1/pcp-atop.1.html),
[pcp-htop(1)](https://man7.org/linux/man-pages/man1/pcp-htop.1.html),
[pmrep(1)](https://man7.org/linux/man-pages/man1/pmrep.1.html), and
_cache*_, _cpu*_, _execsnoop_, _off*_, _run*_ commands from the
_bcc-tools_ package.

#### Per-Process CPU Monitoring

```
# Report 5 most CPU using processes
pmrep -1gU -t 2 -J 5 proc.hog.cpu
# Report process CPU/scheduling statistics
pmrep -gp -t 2 -i '.*PROCNAME.*' :proc-cpu :proc-cpu-ext
# Show process CPU affinity and priority information
tuna -t PROCNAME -P
# Report process CPU usage details
perf top -Kgnv -d 2 -e cycles:u -s cpu,pid,comm,dso,sym -p PID
# Report process user-space CPU cycle and cache stats
perf stat -e cycles:u -d -I 2000 -p PID
# Report process scheduler event stats
perf stat -e 'sched:*' -I 2000 -p PID 2>&1 \
  | grep -v " 0 " | grep -v "not counted"
```

```
# Testing
linpack ...
lmbench ...
sysbench ...
```

## Memory Tuning and Monitoring

Memory related documentation references:

* [RHEL Configuring huge pages guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/assembly_configuring-huge-pages_monitoring-and-managing-system-status-and-performance)
* [RHEL Configuring an operating system to optimize memory access guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/assembly_configuring-an-operating-system-to-optimize-memory-access_monitoring-and-managing-system-status-and-performance)
* [RHEL Profiling memory allocation with numastat guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/profiling-memory-allocation-with-numastat_monitoring-and-managing-system-status-and-performance)
* [RHEL Profiling memory accesses with perf mem guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/profiling-memory-accesses-with-perf-mem_monitoring-and-managing-system-status-and-performance)
* [RHEL Detecting false sharing with perf c2c guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/detecting_false_sharing_with_perf_c2c)
* [RHEL Performance Analysis and Tuning PDF](https://www.redhat.com/cms/managed-files/Handout%20Performance%20Analysis%20and%20Tuning%20Red%20Hat%20Enterprise%20Linux%202019.pdf)
* [In defence of swap: common misconceptions blog post](https://chrisdown.name/2018/01/02/in-defence-of-swap.html)
* [Linux kernel VM sysctl document](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)
* [RHEL technology capabilities and limits page](https://access.redhat.com/articles/rhel-limits)
* [Linux Performance page by Brendan Gregg](https://www.brendangregg.com/linuxperf.html)

### Hardware and System-wide Memory Configuration

There are usually not many memory related hardware configurations but it
is a good idea to check Memory Protection which should be Enabled and
also make sure Node Interleaving is Disabled.

The suitable amount of swap depends on the use case but a good starting
point could be 4 GB. Having no swap at all rarely provides for the best
performance and using huge amounts of swap is usually not helpful.

The Linux kernel uses memory not in use by applications for buffering
and caching. While buffering and caching is a good thing constant paging
and swapping is not and extreme swapping can render the system almost
completely unresponsive. The most important swapping metric is swapping
activity, that is, how much pages are being swapped in and out, not the
plain amount of swap currently in use. There might be a portion of swap
in use at any given time but in case there is no constant swapping
activity then this swap usage is a merely a sign that there has been a
memory pressure situation in the past and the kernel has paged out some
idle pages or processes to make room for actively running applications,
or perhaps for buffering and caching. Since all modern operating systems
use demand paging the swapped out pages are not proactively swapped back
into the main memory until there is a real need for them so swap may
remain long used after a memory pressure situation.

See the following articles for more discussion on swap:

* [What is the recommended swap size for Red Hat platforms?](https://access.redhat.com/solutions/15244)
* https://chrisdown.name/2018/01/02/in-defence-of-swap.html
* https://www.redhat.com/en/blog/do-we-really-need-swap-modern-systems

In case the system runs out of memory the dreaded OOM-killer will act
(the per-process oom related tunables are described in the second link):

* [How does the OOM-Killer select a task to kill?](https://access.redhat.com/solutions/66458)
* https://www.kernel.org/doc/Documentation/filesystems/proc.txt
* https://man7.org/linux/man-pages/man8/systemd-oomd.service.8.html

```
# Show current memory layout
lsmem -o +NODE -S NODE
# Show compact NUMA layout
numactl -H
# Show current memory usage
free -m
```

### System V IPC

Shared memory, semaphore, and message queue configuration on the system
should be changed when applications require more of these resources than
is available by default. Please refer to application documentation for
their exact requirements.

* [What are the kernel parameters for IPC?](https://access.redhat.com/solutions/20519)
* [What are the kernel parameters available for System V IPC tuning?](https://access.redhat.com/solutions/431633)
* [Changing the semaphore value fails with large values](https://access.redhat.com/solutions/4968021)
* https://www.kernel.org/doc/Documentation/sysctl/kernel.txt

```
# Show current System V IPC kernel parameters
sysctl -a | grep -Ei 'kernel.(shma|shmm|sem |msgm)'
# Show current System V IPC status and usage
ipcs -u
ipcs -a
```

### Huge Pages

Transparent Huge Pages (THP) are enabled by default on RHEL and are
usually helpful with most applications. However, some memory heavy
applications such as databases often benefit from static huge pages
especially on very large memory systems. Some applications (or at least
certain application versions) have [higher CPU usage with
THP](https://access.redhat.com/solutions/1265183).

Considering the above, it is best to check the configuration
recommendations of each application and then measure the results of
different approaches. Depending on the use case and workload profile
(e.g., latency or throughput sensitive) either THP, possibly customized
khugepaged parameters, or static huge pages setup may yield the best
performance. Exotic setups like NUMA node specific huge page
configurations are rarely helpful. Note that some application vendors
recommend disabling huge pages altogether; please refer to vendor
documentation for the exact recommendations.

* [How can I configure hugepages in RHEL?](https://access.redhat.com/solutions/33613)
* [How to use, monitor, and disable transparent hugepages in RHEL?](https://access.redhat.com/solutions/46111)
* [How do I check for hugepages usage and what is using it?](https://access.redhat.com/solutions/320303)
* [Examining Huge Pages or Transparent Huge Pages performance](https://access.redhat.com/articles/781653)
* https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt
* https://alexandrnikitin.github.io/blog/transparent-hugepages-measuring-the-performance-impact/

```
# Show current Huge Page statistics
grep Huge /proc/meminfo
# Show current THP configuration
grep -H . /sys/kernel/mm/transparent_hugepage/*
# Show current khugepaged configuration
grep -H . /sys/kernel/mm/transparent_hugepage/khugepaged/*
```

### NUMA

NUMA (non-uniform memory access) setup can be crucial on larger systems.
On virtual platforms it should also be made sure NUMA topology awareness
is propagated properly by CPU pinning in case applications are running
on large VMs spanning over several NUMA nodes.

By default automatic kernel NUMA balancing is used. This is suitable in
the majority of use cases but certain applications (like databases) may
benefit from disabling this feature. Again, please refer to application
specific documentation for further details and recommendations on this.

<pre>
# Show current NUMA usage
numastat -cmz
# Show current NUMA statistics
numastat -n
# Show current kernel NUMA balancing parameters
# <a href="https://access.redhat.com/solutions/1533463">Automatic NUMA balancing in RHEL</a>
grep -H . /proc/sys/kernel/numa*
</pre>

In some relatively rare cases using
[numad(8)](https://www.mankier.com/8/numad) instead of the kernel
automatic NUMA balancing might provide some performance benefits,
however this should be used only when measured to have positive
performance impact. Sometimes static NUMA bindings can outperform both
automatic and [numad(8)](https://www.mankier.com/8/numad) based
balancing, this should be considered only for very specific workloads
(such as certain VNFs). See
[taskset(1)](https://man7.org/linux/man-pages/man1/taskset.1.html) and
[numactl(8)](https://man7.org/linux/man-pages/man8/numactl.8.html) for
details on CPU affinity and static NUMA bindings.

* https://www.redhat.com/en/blog/mysteries-numa-memory-management-revealed

### Memory Related Kernel Parameters

See the above sections about IPC, THP, and NUMA related parameters.

Many of the tuned profiles set these up properly for most cases.

Do NOT change any of these parameters without testing and understanding
their meaning, they might cause negative performance impact especially
under heavy load if changed inappropriately.

<pre>
# <a href="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/assembly_configuring-an-operating-system-to-optimize-memory-access_monitoring-and-managing-system-status-and-performance#assembly_optimizing-a-systems-memory-utilization_assembly_configuring-an-operating-system-to-optimize-memory-access">RHEL 8 documentation section</a>
# <a href="https://www.kernel.org/doc/Documentation/sysctl/vm.txt">https://www.kernel.org/doc/Documentation/sysctl/vm.txt</a>
vm.dirty_ratio = N
vm.dirty_background_ratio = N
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/103833">What does swappiness do?</a>
# <a href="https://access.redhat.com/solutions/3383811">How swappiness sysctl works in RHEL?</a>
# <a href="https://www.kernel.org/doc/Documentation/sysctl/vm.txt">https://www.kernel.org/doc/Documentation/sysctl/vm.txt</a>
vm.swappiness = N
vm.vfs_cache_pressure = N
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/2209921">What is kernel parameter vm.zone_reclaim_mode?</a>
vm.zone_reclaim_mode = 1
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/336033">What is the vm.min_free_kbytes parameter?</a>
vm.min_free_kbytes = N
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/68612">What is the vm.overcommit_memory parameter?</a>
vm.overcommit_memory = N
</pre>

### Configuring Applications and Services

The most important configuration is to have enough memory available for
running applications and services to avoid constant swapping or OOM. For
details on IPC, THP, and NUMA configuration for each application please
refer to their documentation.

### Memory Related Monitoring and Testing

#### System-wide Memory Monitoring

```
# Monitor memory statistics
top
sar -W 2
sar -B 2
sar -H 2
sar -r ALL 2
vmstat -S m -w 2
# Report kernel PSI memory metrics
pmrep -gp -t 2 kernel.all.pressure.memory
# Report NUMA related statistics
pmrep -p -t 2 :numa-hint-faults
pmrep -p -t 2 :numa-pgmigrate-per-node
# Report various memory related metrics
pmrep mem<TAB>
```

See also [PCP](https://pcp.io/) tools
[pcp-atop(1)](https://man7.org/linux/man-pages/man1/pcp-atop.1.html),
[pcp-htop(1)](https://man7.org/linux/man-pages/man1/pcp-htop.1.html),
[pmrep(1)](https://man7.org/linux/man-pages/man1/pmrep.1.html), and
_cache*_, _oomkill_, _shmsnoop_, _swapin_ commands from the _bcc-tools_
package.

#### Per-Process Memory Monitoring

```
# Report 5 most memory using processes
pmrep -1gU -t 2 -J 5 proc.hog.mem
# Report process memory usage statistics
pmrep -gp -t 2 -i '.*PROCNAME.*' :proc-mem :proc-mem-ext
# Report process per-NUMA-node memory statistics
watch -d=cumu -n 2 numastat -c PID
# Summary of memory related system calls
strace -fc -e trace=%memory COMMAND
# Report page migrations
perf stat -e migrate:mm_migrate_pages -I 2000 -p PID
# Report page faults
perf stat -e page-faults -I 2000 -p PID
```

```
# Trace memory accesses
perf mem record -a -- sleep 10
perf mem report -s mem
perf mem report -s cpu,pid,comm,tlb,mem
```

<pre>
# Trace false cache sharing,
# consider higher -F values as per the blog post for more details
# <a href="https://joemario.github.io/blog/2016/09/01/c2c-blog/">https://joemario.github.io/blog/2016/09/01/c2c-blog/</a>
perf c2c record -F max -u -a -- sleep 5
perf c2c report -NN -c pid,dso --stdio
</pre>

```
# Testing
lmbench ...
stress ...
```

## IO Tuning and Monitoring

IO related documentation references:

* [RHEL Factors affecting I/O and file system performance guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/factors-affecting-i-o-and-file-system-performance_monitoring-and-managing-system-status-and-performance)
* [RHEL Configuring and managing logical volumes guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/index)
* [RHEL Configuring device mapper multipath guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_device_mapper_multipath/index)
* [RHEL Deduplicating and compressing storage guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/deduplicating_and_compressing_storage/index)
* [RHEL Deduplicating and compressing logical volumes on RHEL guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/deduplicating_and_compressing_logical_volumes_on_rhel/index)
* [Linux kernel VM sysctl document](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)
* [RHEL technology capabilities and limits page](https://access.redhat.com/articles/rhel-limits)
* [Linux Performance page by Brendan Gregg](https://www.brendangregg.com/linuxperf.html)

### Block Device and File System Layout and Details

```
lsblk
lsblk -t
pvs
vgs
lvs
df -hT
df -hiT
findmnt
dmesg -T
dmsetup info
dmsetup table
multipath -ll
dumpe2fs -h /dev/sdX1
xfs_info /dev/sdX1
```

### Block Device Preparation and Parameters

Ensure correct RAID level and disk type specific BIOS/HBA parameters are
in use. For advanced options like LVM striping multipath options, and
VDO see the above listed Red Hat documentation.

<pre>
# <a href="https://access.redhat.com/solutions/5427">What is the suggested I/O scheduler for RHEL with virtualization?</a>
echo N > /sys/block/DEVICE/queue/scheduler
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/43861">What is the kernel parameters related to maximum size of physical I/O requests?</a>
# Should be aligned with optimal_io_size if reported
echo N > /sys/block/DEVICE/queue/max_sectors_kb
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/4378581">How to set the read_ahead_kb value persistently?</a>
# Can be up to max_sectors_kb
echo N > /sys/block/DEVICE/queue/read_ahead_kb
</pre>

### File System Creation and Mount Parameters

<pre>
# For lots of small files test smaller block size
# <a href="https://access.redhat.com/solutions/37930">Filesystem with a blocksize larger than 4096?</a>
# <a href="https://access.redhat.com/solutions/1614393">What is the maximum supported XFS block size in RHEL?</a>
# <a href="https://access.redhat.com/solutions/3202781">How to create XFS filesystem with 512 bytes block size?</a>
# <a href="https://access.redhat.com/solutions/70522">What is the maximum block size of the ext4 filesystem in RHEL?</a>
mkfs.xfs -b size=1024 /dev/sdX1
mkfs.ext4 -b 1024 /dev/sdX1
</pre>

<pre>
# Avoid excessive reserved space with huge ext4 partitions
# <a href="https://access.redhat.com/solutions/2076">df shows 5% less free space than is present on a fresh filesystem?</a>
# <a href="https://access.redhat.com/solutions/1980673">Does XFS have reserved space for the root user?</a>
mke2fs -j -m 2 /dev/sdX1
tune2fs -m 2 /dev/sdX1
</pre>

<pre>
# Ensure one of relatime or nodiratime or noatime is set
# <a href="https://opensource.com/article/20/6/linux-noatime">https://opensource.com/article/20/6/linux-noatime</a>
findmnt | grep -v atime
</pre>

### Kernel VM IO Related Parameters

tuned profiles set these up properly for most cases.

<pre>
# <a href="https://www.kernel.org/doc/Documentation/sysctl/vm.txt">https://www.kernel.org/doc/Documentation/sysctl/vm.txt</a>
vm.dirty_ratio = N
vm.dirty_background_ratio = N
vm.swappiness = N
</pre>

### Configuring Applications and Services

Consider using dedicated partitions as needed and per vendor
recommendations. For recommendations for applications please refer to
vendor documentation.

### IO Related Monitoring and Testing

#### System-wide IO Monitoring

```
# Monitor IO statistics
sar -bd 2
vmstat -S m -w 2
iotop -Poka -d 2
# Report kernel PSI IO metrics
pmrep -gp -t 2 kernel.all.pressure.io
# Monitor block device activity
iostat -dmtxz -p ALL 2
btrace -t -s /dev/sdX1
# Report various FS/IO related metrics
pmrep filesys<TAB>
pmrep mounts<TAB>
pmrep disk<TAB>
pmrep vfs<TAB>
```

See also [PCP](https://pcp.io/) tools
[pcp-atop(1)](https://man7.org/linux/man-pages/man1/pcp-atop.1.html),
[pcp-htop(1)](https://man7.org/linux/man-pages/man1/pcp-htop.1.html),
[pmrep(1)](https://man7.org/linux/man-pages/man1/pmrep.1.html), and
_bio*_, _file*_, _{ext4,xfs}slower_, _opensnoop_ commands from the
_bcc-tools_ package.

#### Per-Process IO Monitoring

```
iotop -k -d 2 -p PID
pidstat -h -d -r -u -w -p PID 2
# Report 5 most disk using processes
pmrep -1gU -t 2 -J 5 proc.hog.disk
# Report process IO statistics
pmrep -gp -t 2 -i '.*PROCNAME.*' :proc-io :proc-io-ext
# List open files and directories
lsof -VanP -r 2 -p PID | grep -e REG -e DIR
# Trace file handling related system calls
strace -CfttTy -e trace=%file,%desc -p PID
```

```
# Testing
# Before testing file system caches must be dropped with:
sync ; echo 3 > /proc/sys/vm/drop_caches ;
dd if=/dev/zero of=/var/tmp/dd-test conv=fsync bs=N count=N
fio test.fio
```

## Network Tuning and Monitoring

Network related documentation references:

* [RHEL Configuring and managing networking guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/index)
* [RHEL Configuring RHEL to optimize access to network resources guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/configuring-rhel-to-optimize-access-to-network-resources_monitoring-and-managing-system-status-and-performance)
* [Illustrated Guide to Monitoring and Tuning the Linux Networking Stack blog post](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/)
* [Monitoring and Tuning the Linux Networking Stack blog post](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/)
* [Linux kernel IP sysctl documentation](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)
* [Linux kernel network scaling documentation](https://www.kernel.org/doc/Documentation/networking/scaling.txt)
* [RHKB How does tcpdump capture packets? article](https://access.redhat.com/solutions/4272142)
* [Linux Performance page by Brendan Gregg](https://www.brendangregg.com/linuxperf.html)

### NIC and System-wide Network Configuration

* [RHEL network interface dropping packets](https://access.redhat.com/solutions/21301)
* [Should I be concerned about a 0.05% packet drop rate?](https://access.redhat.com/solutions/742043)
* [ifconfig and ip commands report packet drops?](https://access.redhat.com/solutions/2073223)
* [What does each field of the "netstat -s" command mean?](https://access.redhat.com/solutions/419803)

```
# Show NIC details and counters
ip -s -s link show [or ip -s -s l]
# Show NIC addresses
ip address show [or ip a]
# Show NIC driver details
ethtool -i eth0
# Show NIC statistics
ethtool -S eth0
# Show routing information
ip route show [or ip r]
# Show ARP information
ip neigh show [or ip n]
# Report various NIC metrics
pmrep network.interface<TAB>
```

### Jumbo Frames

Consider Jumbo frames where suitable, see
[How to enable jumbo frames for network interfaces in RHEL?](https://access.redhat.com/solutions/3643).

### NIC Parameters

<pre>
# <a href="https://access.redhat.com/solutions/6381">How do I modify txqueuelen of my NIC?</a>
ip link set eth0 txqueuelen 2000
ethtool -g eth0
ethtool -G eth0 rx 4096
ethtool -G eth0 tx 4096
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/288433">What do the offload parameters shown by ethtool -k mean?</a>
ethtool -k eth0
ethtool -K eth0 ...
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/4367191">How should I configure network interface IRQ channels?</a>
ethtool -l eth0
ethtool -L eth0 combined N
</pre>

<pre>
# <a href="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/assembly_configuring-ethtool-coalesce-settings_configuring-and-managing-networking">RHEL 8 documentation section</a>
ethtool -c eth0
ethtool -C eth0 adaptive-rx on
ethtool -C eth0 adaptive-tx on
</pre>

### Various Network Stack Parameters

<pre>
# <a href="https://access.redhat.com/solutions/85913">What are the implications of changing socket buffer sizes?</a>
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/168483">How do I tune RHEL for better TCP performance?</a>
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 87380 16777216
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/3713681">How to configure TCP BBR congestion control algorithm?</a>
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/168483">How do I tune RHEL for better TCP performance?</a>
net.ipv4.tcp_slow_start_after_idle = 0
</pre>

<pre>
# <a href="https://sysctl-explorer.net/net/ipv4/tcp_fastopen/">https://sysctl-explorer.net/net/ipv4/tcp_fastopen/</a>
net.ipv4.tcp_fastopen = 3
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/30453">Kernel SYN flood messages explained</a>
net.core.somaxconn = 4096
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/31043">Possible values for tcp_retries2 and tcp_syn_retries</a>
net.ipv4.tcp_syn_retries = 5
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/1241943">How to tune net.core.netdev_max_backlog kernel tunable?</a>
net.core.netdev_max_backlog = 1000
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/1241943">How to tune net.core.netdev_budget kernel tunable?</a>
net.core.netdev_budget = 600
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/3642462">Is it safe to disable net.ipv4.tcp_timestamps?</a>
net.ipv4.tcp_timestamps = 0
</pre>

<pre>
# <a href="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-networking-configuration_tools#sect-Red_Hat_Enterprise_Linux-Performance_Tuning_Guide-Configuration_tools-Configuring_interrupt_queues">RHEL 7 documentation section</a>
# <a href="https://www.kernel.org/doc/Documentation/sysctl/net.txt">https://www.kernel.org/doc/Documentation/sysctl/net.txt</a>
ethtool -k eth0 | grep busy
net.core.busy_read = 50
net.core.busy_poll = 50
</pre>

<pre>
# <a href="https://access.redhat.com/solutions/4978771">TCP F-RTO performance considerations</a>
net.ipv4.tcp_frto = 0
</pre>

```
# Other considerations
net.ipv4.ip_local_port_range = 12288 65535
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_abort_on_overflow = 1
net.ipv4.tcp_no_metrics_save = 1
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_ecn = 1
```

### Additional Optimizations

Consider utilizing RSS / RPS / RFS where needed and supported, see
[RHEL Configuring RHEL to optimize access to network resources guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/configuring-rhel-to-optimize-access-to-network-resources_monitoring-and-managing-system-status-and-performance).

### Configuring Applications and Services

For any local communication always use the localhost address only, not
the public IP or FQDN, both for performance and security reasons.

### Network Related Monitoring and Testing

#### System-wide Network Monitoring

```
# Show network stack summary
ss -s
# Show network stack details
nstat -a
# Monitor network traffic
iptraf-ng
sar -n DEV 2
sar -n SOCK 2
iptstate -lt -R 2
# Show dropped packets
nstat -az | grep -i -e drop -e noroute
# Show socket details
ss -noemitaup
ss -lnptu
ss -nrt
```

See also [PCP](https://pcp.io/) tools
[pcp-atop(1)](https://man7.org/linux/man-pages/man1/pcp-atop.1.html),
[pcp-htop(1)](https://man7.org/linux/man-pages/man1/pcp-htop.1.html),
[pmrep(1)](https://man7.org/linux/man-pages/man1/pmrep.1.html) with [PCP
BCC PMDA netproc
module](https://man7.org/linux/man-pages/man1/pmdabcc.1.html), and
_tcp*_ commands from the _bcc-tools_ package.

#### Per-process Monitoring

```
# Monitor per-port/pid network connections
lsof -VP -r 2 -i :PORT
lsof -VanPi -r 2 -p PID
# Report 5 most network using processes
# Requires PCP BCC PMDA netproc module installed (pcp-5.2.3)
pmrep -1gU -t 2 -J 5 proc.hog.net
# Process network usage statistics
# Requires PCP BCC PMDA netproc module installed (pcp-5.2.3)
# Requires default pmrep.conf update for netproc (pcp-5.3.0)
pmrep -gp -t 2 -i '.*PROCNAME.*' :proc-net :proc-net-ext
# Dump network traffic
tcpdump -nnv -i any port 80 or port 443
# Trace network related system calls
strace -CfttTy -e trace=%network -p PID
```

```
# Testing
curl -v ...
iperf -V ...
```

## Additional Information

* [Systems Performance: Enterprise and the Cloud book](https://www.brendangregg.com/systems-performance-2nd-edition-book.html)
* [Linux Performance page by Brendan Gregg](https://www.brendangregg.com/linuxperf.html)
* [Visualizing performance in RHEL](https://www.redhat.com/en/blog/visualizing-performance-red-hat-enterprise-linux-84-web-console)
* [RHEL Troubleshooting Guide](https://github.com/myllynen/rhel-troubleshooting-guide)
* [RHEL documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/)
* [Red Hat Knowledge Base](https://access.redhat.com/knowledgebase/)
* [Red Hat Customer Portal](https://access.redhat.com/)
