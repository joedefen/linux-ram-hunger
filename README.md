# Linux RAM Hunger
Do you suspect your Linux system is running out of memory?  These notes are intend to:
* help determine if there is a memory issue, and
* if so, how to address it.

This is a humble attempt to modernize [Help! Linux ate my RAM!](https://www.linuxatemyram.com/)

# Localize the Problem
Generally, you might visit here if your systems is sluggish and you suspect a memory issue.
Firstly, determine whether sluggishness (or tanking) it is due memory pressure or something else.

## Am I Running Out of RAM?
Referring to the output of `free -h`:
```
$ free -h
            total     used     free shared buff/cache available
Mem:         31Gi    4.6Gi    6.0Gi  289Mi       20Gi      25Gi
Swap:       8.0Gi     10Mi    8.0Gi
```

generally, the best indicators of inadequate memory for the workload are:
* "available" approaches 0, or
* `dmesg | grep oom-killer` shows the out-of-memory-killer at work, or
* if disk swapping, "swap used" increases or fluctuates.

*Note*: using ZFS (and perhaps other features) may cause "available" to be understated.

**Are you swapping to disk?** To know, run `swapon` and disregard "zram"; e.g., this indicates no swap to disk:
```
$ swapon
NAME       TYPE      SIZE  USED PRIO
/dev/zram0 partition   8G 10.3M  100
```

## Am I Bottlenecked on CPU or Disk?
If your system is slow but not out of ram, run `iostat 10`, to get snapshot of CPU and disk use; e.g.,
```
$ iostat 10
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          16.68    0.00    4.96    0.08    0.00   78.29

Device      tps kB_read/s kB_wrtn/s kB_dscd/s kB_read kB_wrtn kB_dscd
nvme0n1   50.30      0.00    845.60      0.00       0    8456       0
zram0      0.00      0.00      0.00      0.00       0       0       0
```
Observations:
* `%idle` approaching 0 would indicate CPU bound
* `$iowait` well above 0 would indicate I/O bound
* `tps` approaching the capacity of the device also indicates I/O bound (e.g., 100 tps on a mechanical hard drive suggest I/O bound).

For a more complete explanation of `iostat`, visit [Understanding iostat · vaneyckt.io](https://vaneyckt.io/posts/understanding_iostat/). And if you are bottlenecked on anything except memory, then find another resource if necessary.

---
---
---

# Fixing an Out-of-RAM Condition
To address an out-of-ram condition, the primary choices are some combination of:
* reduce your workload and memory demand,
* increase your physical memory, or
* increase your use of zRAM.

## Reducing Your Memory Demand
To reduce the memory demand on your system, it is wise to locate the memory pigs.
You can use top and htop, but I find those tools to be very iffy and often unhelpful.
Instead, I use the proportional memory tool, [pmemstat](https://github.com/joedefen/pmemstat),
for the analysis. Proportional memory metrics are more accurate than say, RSS, and `pmemstat`
rolls up memory use nicely (e.g., combining all the `firefox` processes).

## Increasing Your zRAM Potential
Proper use of zRAM can more than double your effective RAM using compression w/o using any disk swap. Once you start swapping to disk, your system can go into the toilet and slowly or never come back even if the workload abates.

<u>Tips for configuration/using of zRAM</u>:
* **disable any disk-based swap.**  In practice, you never want to swap to disk much anyhow; zRAM is so effective, you should not need to have any disk swap. See "Removing traditional swap partitions and files" in [Make swap better with zRAM on Linux | Opensource.com](https://opensource.com/article/22/11/customize-zram-linux)
* **enable zRAM.**  [Enable Zram on Linux For Better System Performance](https://fosspost.org/enable-zram-on-linux-better-system-performance/) proides instructions for several distros.
* **set your zram "DISKSIZE" to 150% of physical memory** (e.g., 6GB zRAM DISKSIZE for 4GB of physical RAM providing an effect RAM size of about 10GB) per [linux - Why does zram occupy much more memory compared to its "compressed" value? - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/594817/why-does-zram-occupy-much-more-memory-compared-to-its-compressed-value).
* **set your "VM" parameters** per [zRAM - ArchWiki](https://wiki.archlinux.org/title/Zram) as:
    * vm.swappiness = 180
    * vm.watermark_boost_factor = 0
    * vm.watermark_scale_factor = 125
    * vm.page-cluster = 0
* **use `zramctl` to get more detail on zRAM use**. `swapon` provides the basics metrics; `zramctl` provides more if needed.
* **tuning zRAM may be useful/necessary** especially for odd use cases.
