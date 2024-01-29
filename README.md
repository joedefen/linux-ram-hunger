# Linux RAM Hunger
Do you suspect your Linux system is running out of memory?  These notes are intend to:
* help determine if there is a memory issue, and
* if so, how to address it.

This is a humble attempt to modernize [Help! Linux ate my RAM!](https://www.linuxatemyram.com/)

---

# Localize the Problem
Generally, you might visit here if your systems is sluggish and you suspect a memory issue. Firstly, determine whether sluggishness (or tanking) it is due memory pressure or something else.

## Am I Running Out of RAM?
Referring to the output of `free -h`:
```
$ free -h
          total      used      free    shared  buff/cache   available
Mem:      3.9Gi     3.3Gi     252Mi      31Mi       790Mi       670Mi
Swap:     7.8Gi     2.1Gi     5.7Gi
```

generally, the best indicators of inadequate memory for the workload are:
* "available" approaches 0, or
* `dmesg | grep oom-killer` shows the out-of-memory-killer at work, or
* if disk swapping, "swap used" increases or fluctuates or is significant.

*Notes*:
* "available memory" is memory that is free or easily made free to run new apps or allow them more RAM; when near zero, then the system may seriously degrade, start killing apps, and/or freeze.  Usually, memory is made free by reducing the disk cache when can impact performance itself.
* the OOM killer runs when you are out of both RAM and swap, 
* in the example above and if disk swapping, swapping 50% of total would indicate serious memory pressure and likely slow down due to swap.
* using ZFS, zRAM, and other features causes "available" to be understated.
* "used" per `free` has changed several times (run `man free` to check):
  *  `used = total - available` since about mid-2023 (as seen above)
  *  `used = total - free - buff/cache` since perhaps 2012
  *  `used = total - free` from before perhaps 2012

**Are you swapping to disk?** To know, run `swapon` (or `/usr/sbin/swapon` if `/usr/sbin` is not your `$PATH`) and disregard "zram"; e.g., this indicates no swap to disk:
```
$ swapon
NAME       TYPE      SIZE USED PRIO
/dev/zram0 partition 7.8G 1.8G  100

```

## Am I Bottlenecked on CPU or Disk?
**Checking "top"**: When the system is slow, check if actually out of CPU, run top and if idle (`id` on the CPU row) is under, say 50%, you may be CPU bound (and the less idle, the worse); e.g. 81.5% idle is OK in this example:
```
top - 11:00:38 up 57 min,  2 users,  load average: 0.49, 0.64, 0.53
Tasks: 233 total,   1 running, 232 sleeping,   0 stopped,   0 zombie
%Cpu(s): 14.9 us,  3.2 sy,  0.0 ni, 81.5 id,  0.0 wa,  0.0 hi,  0.2 si,  0.2 st 
MiB Mem :   4003.2 total,    219.9 free,   3236.7 used,    939.2 buff/cache     
MiB Swap:   8006.4 total,   5464.9 free,   2541.5 used.    766.5 avail Mem 
```
As a bonus, `top` accurately shows periodically updated `free` numbers (but in megabytes by default).

**Checking iostat**: For more detail, run `iostat 10`, to get snapshot of CPU and disk use; e.g.,
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

For a more complete explanation of `iostat`, visit [Understanding iostat · vaneyckt.io](https://vaneyckt.io/posts/understanding_iostat/). And if you are bottlenecked on anything except memory, then find another resource if necessary.  You may need to install `iostat` (e.g., `sudo apt install sysstat` on Debian).

---

# Fixing an Out-of-RAM Condition
To address an out-of-ram condition, the primary choices are some combination of:
* reduce your workload and memory demand,
* increase your physical memory, or
* increase your use of zRAM or disk swap.

## Reducing Your Memory Demand
To reduce the memory demand on your system, it is wise to locate the memory pigs. You can use top and htop, but I find those tools to be very iffy and often unhelpful. Instead, I use the proportional memory tool, [pmemstat](https://github.com/joedefen/pmemstat), for the analysis. Proportional memory metrics are more accurate than say, RSS, and `pmemstat` rolls up memory use nicely (e.g., combining all the `firefox` processes).

In many case, your browser will be the memory hog; Chrome's Memory Saver feature and extensions like "Auto Tab Discard" can reduce browser memory by unloading tabs not actively in use.

## Increasing Your zRAM Potential
Proper use of zRAM can more than double your effective RAM using compression w/o using any disk swap. zRAM is especially helpful on systems with low memory such as 2GB, 4GB or 8GB of RAM AND with slower disks; with more RAM, it will help only if RAM is still inadequate.  zRAM costs CPU however, and sometimes it is actually better to swap to disk (e.g., swapping to NVMe disks).

***Tips for configuration/using of zRAM***:
* **optionally, disable any disk-based swap.**  When zRAM, you do not want to swap to disk, but it suffices to just set its swap priority higher than the disk swap. See "Removing traditional swap partitions and files" in [Make swap better with zRAM on Linux | Opensource.com](https://opensource.com/article/22/11/customize-zram-linux). Note:
   * There are reasons to configure both zRAM and a disk swap area (e.g., to configure hibernation, to permit even more virtual memory than allowed by zRAM alone, etc.) For that niche, seek appropriate guides.
<br>

* **enable zRAM.**  [Enable Zram on Linux For Better System Performance](https://fosspost.org/enable-zram-on-linux-better-system-performance/) provides instructions for several distros. Examples:
  * Fedora, Pop!_OS, and Endless OS: enabled on install.
  * Linux-Lite: enable with "Lite Tweaks" app.
  * Debian 12: `sudo apt install zram-tools`
<br>

* **set your zram "DISKSIZE" to 150% of physical memory (nominally)** (e.g., 6GB zRAM DISKSIZE for 4GB of physical RAM providing an effect RAM size of about 10GB) per [linux - Why does zram occupy much more memory compared to its "compressed" value?](https://unix.stackexchange.com/questions/594817/why-does-zram-occupy-much-more-memory-compared-to-its-compressed-value). Again, see in [Enable Zram on Linux For Better System Performance](https://fosspost.org/enable-zram-on-linux-better-system-performance/) for how to set zRAM size for several distros. Examples:
 
  * Fedora:  edit `/usr/lib/systemd/zram-generator.conf` and set `zram-size = ram * 1.5`
  * Linux-Lite: edit `/usr/bin/init-zram-swapping` and set `mem=$((totalmem*2*1024))`
  * Debian 12: edit `/etc/default/zramswap` and uncomment/set PERCENT=150, and run `sudo systemctl enable --now zramswap`

>> * Don't gratuitously configure more zRAM than you'd ever use because that adds some inefficiencies w/o benefit; e.g., if using 24GB is unimaginable having 16GB physical, then cap zRAM at 8GB.
>> * But, on the lower end, 200% of RAM may be best (which is the amount Chromebooks configure.)

<br>

* **set your "VM" parameters** per [zRAM - ArchWiki](https://wiki.archlinux.org/title/Zram) as:
```
    # add/replace these lines in /etc/sysctl.conf
    vm.swappiness = 180
    vm.watermark_boost_factor = 0
    vm.watermark_scale_factor = 125
    vm.page-cluster = 0
```

* **reboot after making zRAM configuration changes** to ensure they take effect.
<br>

## More zRAM Tips
* **use `zramctl` to get more detail on zRAM use**. Compute your compression ratio as COMPR/DATA (so about 4 in the this example and your "DISKSIZE" can be set to `CompressionRatio*RAM/2` for full effect typically or 200% of RAM in this example).
```
    $ zramctl
    NAME       ALGORITHM DISKSIZE DATA  COMPR  TOTAL STREAMS MOUNTPOINT
    /dev/zram0 lz4           7.8G   2G 520.6M 545.1M       2 [SWAP]
```
## Run zram-advisor to Check and Configure zRAM
[zram-advisor](https://pypi.org/project/zram-advisor/) checks your currently running zRAM if any, and if desired, installs `fix-zram` to control and initialize zRAM on boot.
