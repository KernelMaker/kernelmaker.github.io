---
layout: post
title: NVMe-SSD性能测试
---



最近在搞比赛的事情，压测了各种引擎的性能，今天用fio压我们的NVMe-SSD，打算用这个数据结合引擎设计一下我们的比赛标程。

## 1. 测试环境

OS版本：3.10.0-327.***.x86_64

CPU：Intel(R) Xeon(R) CPU E5-2682 v4 @ 2.50GHz

磁盘：Non-Volatile memory controller: Intel Corporation PCIe Data Center SSD (rev 01)（1.8T）

## 2. 测试结果

**【注：均是使用direct io跑的结果】**

BandWidth、IOPS、Latency需要分开测试，各跑出其极限值，因为他们之间会互相干扰，比如在测IOPS时，如果BandWidth先达到瓶颈，那么IPOS会不准确。再如测Latency时，如果BandWidth或IOPS任一先达到瓶颈时也会导致Latency不准确，所以需要使用不同的参数去单独跑测试。

### 2.1 裸盘

|                  | BandWidth | IOPS | Latency(usec)                                              |
| ---------------- | --------- | ---- | ---------------------------------------------------------- |
| Sequential Write | 972MB/s   | 188K | 2=0.01%, 10=0.02%, 【20=97.41%】, 50=2.48%, 100=0.08%      |
| Random Write     | 623MB/s   | 184K | 2=0.01%, 10=0.02%, 【20=93.72%】, 50=5.92%, 100=0.33%      |
| Sequential Read  | 2493MB/s  | 196K | 2=0.01%, 4=0.01%, 10=0.03%,【 20=99.81%】, 50=0.15%        |
| Random Read      | 2494MB/s  | 188K | 50=0.01%, 【100=64.62%】, 250=34.96%, 500=0.20%, 750=0.04% |

### 2.2 EXT4

|                  | BandWidth | IOPS | Latency(usec)                                             |
| ---------------- | --------- | ---- | --------------------------------------------------------- |
| Sequential Write | 1943MB/s  | 193K | 2=0.01%, 4=0.01%, 【10=91.10%】, 20=8.44%, 50=0.46%       |
| Random Write     | 1956MB/s  | 192K | 4=0.01%, 【10=94.51%】, 20=5.22%, 50=0.26%, 100=0.01%     |
| Sequential Read  | 2741MB/s  | 194K | 2=0.01%, 4=0.01%, 【10=99.07%】, 20=0.91%, 50=0.01%       |
| Random Read      | 2704MB/s  | 188K | 50=0.01%, 【100=99.55%】, 250=0.44%, 500=0.01%, 750=0.01% |

**结果比较：**

1. 裸盘write bandwidth整体比read bandwidth少很多
2. 裸盘write bandwidth比EXT4上的write bandwidth少很多
3. 裸盘、EXT4和随机、顺序读写的IOPS差不多
4. 无论裸盘还是EXT4，Random Read的latency相比其他latency高很多



## 3. 测试详情

测试裸盘和在EXT4上的方法，除了参数filename不一样外，其他均一样，所以以测试裸盘为例，列出测试详情

**【呃。。这博客页面貌似支持不了一行太长的字符（我也懒得换行了），命令已经伸出黑框外了。。。需要拷贝命令的话直接用鼠标点选，那样会显示出所有信息】**

### 3.1 BandWidth

测试BandWidth时，bs和iodepth需要设置大一些才可以跑满BandWidth

bs=1m，iodepth=4096（我从iodepth=4开始不断调整观察，直到4096时bandwidth趋于稳定）

【**1. Sequential Write**】

```
sudo fio --name=sequential_write_bandwidth_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=1M --iodepth=4096 --rw=write --group_reporting        

sequential_write_bandwidth_test: (g=0): rw=write, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=4096
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [W(1)][100.0%][r=0KiB/s,w=603MiB/s][r=0,w=603 IOPS][eta 00m:00s]
sequential_write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=10944: Thu Jul 12 16:09:58 2018
  write: IOPS=872, BW=927MiB/s (972MB/s)(54.5GiB/60249msec)
    slat (usec): min=124, max=11720, avg=1139.54, stdev=1007.79
    clat (msec): min=247, max=7027, avg=4476.22, stdev=1759.98
     lat (msec): min=250, max=7028, avg=4477.33, stdev=1760.32
    clat percentiles (msec):
     |  1.00th=[ 1183],  5.00th=[ 2668], 10.00th=[ 2702], 20.00th=[ 2769],
     | 30.00th=[ 2769], 40.00th=[ 2802], 50.00th=[ 4665], 60.00th=[ 5671],
     | 70.00th=[ 5940], 80.00th=[ 6342], 90.00th=[ 6745], 95.00th=[ 6812],
     | 99.00th=[ 6946], 99.50th=[ 7013], 99.90th=[ 7013], 99.95th=[ 7013],
     | 99.99th=[ 7013]
   bw (  KiB/s): min=419096, max=1709402, per=94.23%, avg=894157.55, stdev=391496.27, samples=119
   iops        : min=  409, max= 1669, avg=872.80, stdev=382.28, samples=119
  lat (msec)   : 250=0.01%, 500=0.29%, 750=0.28%, 1000=0.29%, 2000=1.17%
  lat (msec)   : >=2000=104.16%
  cpu          : usr=5.27%, sys=11.27%, ctx=39050, majf=0, minf=20
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=106.1%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwt: total=0,52572,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
  WRITE: bw=927MiB/s (972MB/s), 927MiB/s-927MiB/s (972MB/s-972MB/s), io=54.5GiB (58.5GB), run=60249-60249msec

Disk stats (read/write):
  nvme10n1: ios=60/446640, merge=0/0, ticks=3/64519554, in_queue=64532536, util=96.82%
```

【**2. Random Write**】

```
sudo fio --name=random_write_bandwidth_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=1M --iodepth=4096 --rw=randwrite --group_reporting

random_write_bandwidth_test: (g=0): rw=randwrite, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=4096
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [w(1)][100.0%][r=0KiB/s,w=502MiB/s][r=0,w=502 IOPS][eta 00m:00s]
random_write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=2884: Thu Jul 12 16:07:02 2018
  write: IOPS=540, BW=594MiB/s (623MB/s)(34.0GiB/60297msec)
    slat (usec): min=131, max=25398, avg=1839.18, stdev=1502.31
    clat (msec): min=295, max=9248, avg=6973.32, stdev=2464.73
     lat (msec): min=296, max=9251, avg=6975.05, stdev=2465.17
    clat percentiles (msec):
     |  1.00th=[ 1011],  5.00th=[ 2299], 10.00th=[ 2400], 20.00th=[ 4111],
     | 30.00th=[ 7148], 40.00th=[ 7819], 50.00th=[ 8221], 60.00th=[ 8356],
     | 70.00th=[ 8658], 80.00th=[ 8792], 90.00th=[ 8926], 95.00th=[ 9060],
     | 99.00th=[ 9194], 99.50th=[ 9194], 99.90th=[ 9194], 99.95th=[ 9194],
     | 99.99th=[ 9194]
   bw (  KiB/s): min=67584, max=1835008, per=89.38%, avg=543989.71, stdev=255279.28, samples=120
   iops        : min=   66, max= 1792, avg=530.68, stdev=249.43, samples=120
  lat (msec)   : 500=0.33%, 750=0.37%, 1000=0.39%, 2000=1.53%, >=2000=107.36%
  cpu          : usr=3.29%, sys=6.93%, ctx=26441, majf=0, minf=19
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=109.8%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwt: total=0,32586,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
  WRITE: bw=594MiB/s (623MB/s), 594MiB/s-594MiB/s (623MB/s-623MB/s), io=34.0GiB (37.6GB), run=60297-60297msec

Disk stats (read/write):
  nvme10n1: ios=60/286603, merge=0/0, ticks=3/66858073, in_queue=66897412, util=96.84%
```



【**3. Sequential Read**】

```
sudo fio --name=sequential_read_bandwidth_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=1M --iodepth=4096 --rw=read --group_reporting

sequential_read_bandwidth_test: (g=0): rw=read, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=4096
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [R(1)][100.0%][r=2292MiB/s,w=0KiB/s][r=2292,w=0 IOPS][eta 00m:00s]
write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=127317: Wed Jul 11 20:26:30 2018
   read: IOPS=2309, BW=2378MiB/s (2493MB/s)(139GiB/60065msec)
    slat (usec): min=73, max=2723, avg=431.06, stdev=227.29
    clat (msec): min=64, max=1866, avg=1747.75, stdev=161.79
     lat (msec): min=64, max=1867, avg=1748.18, stdev=161.79
    clat percentiles (msec):
     |  1.00th=[  684],  5.00th=[ 1737], 10.00th=[ 1754], 20.00th=[ 1770],
     | 30.00th=[ 1770], 40.00th=[ 1770], 50.00th=[ 1770], 60.00th=[ 1770],
     | 70.00th=[ 1770], 80.00th=[ 1770], 90.00th=[ 1787], 95.00th=[ 1787],
     | 99.00th=[ 1821], 99.50th=[ 1838], 99.90th=[ 1854], 99.95th=[ 1854],
     | 99.99th=[ 1854]
   bw (  MiB/s): min= 2147, max= 2496, per=97.78%, avg=2325.05, stdev=42.40, samples=120
   iops        : min= 2147, max= 2496, avg=2324.57, stdev=42.35, samples=120
  lat (msec)   : 100=0.06%, 250=0.25%, 500=0.41%, 750=0.42%, 1000=0.41%
  lat (msec)   : 2000=101.41%
  cpu          : usr=0.28%, sys=27.35%, ctx=100846, majf=0, minf=18
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=103.4%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwt: total=138724,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
   READ: bw=2378MiB/s (2493MB/s), 2378MiB/s-2378MiB/s (2493MB/s-2493MB/s), io=139GiB (150GB), run=60065-60065msec

Disk stats (read/write):
  nvme10n1: ios=1146393/0, merge=0/0, ticks=64785071/0, in_queue=64843134, util=99.90%
```



【**4. Random Read**】

```
sudo fio --name=random_read_bandwidth_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=1M --iodepth=4096 --rw=randread --group_reporting

random_read_bandwidth_test: (g=0): rw=randread, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=4096
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [r(1)][100.0%][r=2320MiB/s,w=0KiB/s][r=2320,w=0 IOPS][eta 00m:00s]
write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=2768: Wed Jul 11 20:28:54 2018
   read: IOPS=2310, BW=2378MiB/s (2494MB/s)(140GiB/60062msec)
    slat (usec): min=70, max=3257, avg=430.75, stdev=216.68
    clat (msec): min=60, max=1865, avg=1747.02, stdev=163.26
     lat (msec): min=61, max=1866, avg=1747.45, stdev=163.26
    clat percentiles (msec):
     |  1.00th=[  676],  5.00th=[ 1754], 10.00th=[ 1754], 20.00th=[ 1754],
     | 30.00th=[ 1754], 40.00th=[ 1770], 50.00th=[ 1770], 60.00th=[ 1770],
     | 70.00th=[ 1787], 80.00th=[ 1787], 90.00th=[ 1787], 95.00th=[ 1787],
     | 99.00th=[ 1804], 99.50th=[ 1821], 99.90th=[ 1838], 99.95th=[ 1854],
     | 99.99th=[ 1871]
   bw (  MiB/s): min= 2136, max= 2539, per=97.74%, avg=2324.76, stdev=45.75, samples=120
   iops        : min= 2136, max= 2539, avg=2324.23, stdev=45.75, samples=120
  lat (msec)   : 100=0.06%, 250=0.25%, 500=0.42%, 750=0.42%, 1000=0.42%
  lat (msec)   : 2000=101.38%
  cpu          : usr=0.34%, sys=27.36%, ctx=102519, majf=0, minf=17
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=103.4%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwt: total=138756,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
   READ: bw=2378MiB/s (2494MB/s), 2378MiB/s-2378MiB/s (2494MB/s-2494MB/s), io=140GiB (150GB), run=60062-60062msec

Disk stats (read/write):
  nvme10n1: ios=1147001/0, merge=0/0, ticks=63819489/0, in_queue=63877499, util=99.91%
```

### 3.2 IOPS

测试IOPS时，bs不能大，否则可能BandWidth先达到瓶颈导致IOPS测试不准，另外iodepth需要设置大一些。

bs=4k，iodepth=4096

【**1. Sequential Write**】

```
sudo fio --name=sequential_write_iops_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=4096 --rw=write --group_reporting

sequential_write_iops_test: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=4096
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [W(1)][100.0%][r=0KiB/s,w=739MiB/s][r=0,w=189k IOPS][eta 00m:00s]
sequential_write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=33574: Thu Jul 12 14:46:54 2018
  write: IOPS=188k, BW=736MiB/s (772MB/s)(43.1GiB/60001msec)
    slat (nsec): min=1403, max=3195.4k, avg=4574.21, stdev=4263.56
    clat (usec): min=66, max=32979, avg=21725.85, stdev=741.85
     lat (usec): min=69, max=32986, avg=21730.47, stdev=741.97
    clat percentiles (usec):
     |  1.00th=[20841],  5.00th=[20841], 10.00th=[21103], 20.00th=[21365],
     | 30.00th=[21365], 40.00th=[21365], 50.00th=[21627], 60.00th=[21890],
     | 70.00th=[21890], 80.00th=[22152], 90.00th=[22414], 95.00th=[22676],
     | 99.00th=[22938], 99.50th=[23462], 99.90th=[31327], 99.95th=[32637],
     | 99.99th=[32900]
   bw (  KiB/s): min=705137, max=785730, per=100.00%, avg=757719.83, stdev=16779.90, samples=120
   iops        : min=176284, max=196432, avg=189429.59, stdev=4194.99, samples=120
  lat (usec)   : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.01%, 20=0.09%, 50=99.93%
  cpu          : usr=12.15%, sys=87.81%, ctx=140, majf=0, minf=20
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=103.4%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwt: total=0,11307185,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
  WRITE: bw=736MiB/s (772MB/s), 736MiB/s-736MiB/s (772MB/s-772MB/s), io=43.1GiB (46.3GB), run=60001-60001msec

Disk stats (read/write):
  nvme10n1: ios=60/11692037, merge=0/0, ticks=3/443847, in_queue=442556, util=100.00%
```

【**2. Random Write**】

```
sudo fio --name=random_write_iops_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=4096 --rw=randwrite --group_reporting

fio-3.1
Starting 1 process
Jobs: 1 (f=1): [w(1)][100.0%][r=0KiB/s,w=727MiB/s][r=0,w=186k IOPS][eta 00m:00s]
random_write_iops_test: (groupid=0, jobs=1): err= 0: pid=43330: Thu Jul 12 14:50:38 2018
  write: IOPS=184k, BW=720MiB/s (755MB/s)(42.2GiB/60001msec)
    slat (nsec): min=1370, max=2830.6k, avg=4484.53, stdev=3580.84
    clat (usec): min=68, max=27235, avg=22219.22, stdev=571.94
     lat (usec): min=74, max=27237, avg=22223.75, stdev=572.01
    clat percentiles (usec):
     |  1.00th=[21365],  5.00th=[21365], 10.00th=[21627], 20.00th=[21890],
     | 30.00th=[21890], 40.00th=[21890], 50.00th=[21890], 60.00th=[22152],
     | 70.00th=[22414], 80.00th=[22676], 90.00th=[22938], 95.00th=[23200],
     | 99.00th=[23462], 99.50th=[23725], 99.90th=[25035], 99.95th=[25297],
     | 99.99th=[26346]
   bw (  KiB/s): min=713040, max=766409, per=100.00%, avg=740729.11, stdev=13718.62, samples=120
   iops        : min=178260, max=191602, avg=185182.05, stdev=3429.58, samples=120
  lat (usec)   : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.01%, 20=0.02%, 50=100.00%
  cpu          : usr=15.98%, sys=84.00%, ctx=147, majf=0, minf=19
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=103.3%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwt: total=0,11056137,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
  WRITE: bw=720MiB/s (755MB/s), 720MiB/s-720MiB/s (755MB/s-755MB/s), io=42.2GiB (45.3GB), run=60001-60001msec

Disk stats (read/write):
  nvme10n1: ios=60/11422767, merge=0/0, ticks=4/461560, in_queue=460236, util=100.00%
```



【**3. Sequential Read**】

```
sudo fio --name=sequential_read_iops_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=4096 --rw=read --group_reporting

sequential_read_iops_test: (g=0): rw=read, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=4096
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [R(1)][100.0%][r=764MiB/s,w=0KiB/s][r=196k,w=0 IOPS][eta 00m:00s]
write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=49612: Wed Jul 11 20:45:10 2018
   read: IOPS=196k, BW=767MiB/s (804MB/s)(44.9GiB/60001msec)
    slat (nsec): min=1288, max=75378, avg=4388.97, stdev=2093.74
    clat (usec): min=142, max=24508, avg=20857.54, stdev=252.56
     lat (usec): min=146, max=24516, avg=20861.97, stdev=252.57
    clat percentiles (usec):
     |  1.00th=[20579],  5.00th=[20579], 10.00th=[20841], 20.00th=[20841],
     | 30.00th=[20841], 40.00th=[20841], 50.00th=[20841], 60.00th=[20841],
     | 70.00th=[20841], 80.00th=[20841], 90.00th=[21103], 95.00th=[21103],
     | 99.00th=[21103], 99.50th=[21103], 99.90th=[21890], 99.95th=[22676],
     | 99.99th=[23200]
   bw (  KiB/s): min=785258, max=793824, per=100.00%, avg=789522.24, stdev=1696.98, samples=120
   iops        : min=196314, max=198456, avg=197380.26, stdev=424.30, samples=120
  lat (usec)   : 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.01%, 20=0.02%, 50=100.00%
  cpu          : usr=10.32%, sys=89.78%, ctx=84, majf=0, minf=18
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=103.4%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwt: total=11778091,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
   READ: bw=767MiB/s (804MB/s), 767MiB/s-767MiB/s (804MB/s-804MB/s), io=44.9GiB (48.3GB), run=60001-60001msec

Disk stats (read/write):
  nvme10n1: ios=12177129/0, merge=0/0, ticks=1441382/0, in_queue=1456491, util=100.00%
```



【**4. Random Read**】

```
sudo fio --name=random_read_iops_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=4096 --rw=randread --group_reporting

random_read_iops_test: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=4096
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [r(1)][100.0%][r=736MiB/s,w=0KiB/s][r=188k,w=0 IOPS][eta 00m:00s]
write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=54241: Wed Jul 11 20:46:50 2018
   read: IOPS=188k, BW=733MiB/s (769MB/s)(42.0GiB/60001msec)
    slat (nsec): min=1301, max=268157, avg=4447.96, stdev=1989.39
    clat (usec): min=117, max=29717, avg=21826.15, stdev=1429.96
     lat (usec): min=127, max=29720, avg=21830.64, stdev=1430.18
    clat percentiles (usec):
     |  1.00th=[21103],  5.00th=[21103], 10.00th=[21365], 20.00th=[21365],
     | 30.00th=[21365], 40.00th=[21365], 50.00th=[21365], 60.00th=[21365],
     | 70.00th=[21627], 80.00th=[21627], 90.00th=[21627], 95.00th=[26870],
     | 99.00th=[26870], 99.50th=[27132], 99.90th=[27132], 99.95th=[27132],
     | 99.99th=[28967]
   bw (  KiB/s): min=611975, max=773678, per=100.00%, avg=753928.42, stdev=42484.35, samples=120
   iops        : min=152993, max=193419, avg=188481.77, stdev=10621.11, samples=120
  lat (usec)   : 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.01%, 20=0.02%, 50=100.00%
  cpu          : usr=13.03%, sys=87.07%, ctx=67, majf=0, minf=17
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=103.4%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwt: total=11255264,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
   READ: bw=733MiB/s (769MB/s), 733MiB/s-733MiB/s (769MB/s-769MB/s), io=42.0GiB (46.1GB), run=60001-60001msec

Disk stats (read/write):
  nvme10n1: ios=11642420/0, merge=0/0, ticks=1231892/0, in_queue=1244763, util=100.00%
```

### 3.3 Latency

测试Latency时，bs和iodepth都需要设置小一点，否则如果bandwidth或IPOS先到瓶颈的话，会导致latency不准确

bs=4k，iodepth=1

【**1. Sequential Write**】

```
sudo fio --name=sequential_write_latency_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=1 --rw=write --group_reporting

sequential_write_latency_test: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [W(1)][100.0%][r=0KiB/s,w=218MiB/s][r=0,w=55.9k IOPS][eta 00m:00s]
write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=103083: Wed Jul 11 21:04:29 2018
  write: IOPS=58.6k, BW=229MiB/s (240MB/s)(13.4GiB/60001msec)
    slat (nsec): min=2580, max=66403, avg=3457.38, stdev=424.24
    clat (nsec): min=718, max=6736.0k, avg=13049.72, stdev=11868.72
     lat (usec): min=11, max=6740, avg=16.57, stdev=11.88
    clat percentiles (nsec):
     |  1.00th=[11328],  5.00th=[11456], 10.00th=[11584], 20.00th=[11712],
     | 30.00th=[11840], 40.00th=[11968], 50.00th=[12096], 60.00th=[12352],
     | 70.00th=[12736], 80.00th=[13248], 90.00th=[14528], 95.00th=[16768],
     | 99.00th=[27520], 99.50th=[39168], 99.90th=[49408], 99.95th=[52992],
     | 99.99th=[74240]
   bw (  KiB/s): min=200668, max=252489, per=100.00%, avg=235618.68, stdev=8997.71, samples=120
   iops        : min=50167, max=63122, avg=58904.35, stdev=2249.45, samples=120
  lat (nsec)   : 750=0.01%, 1000=0.01%
  lat (usec)   : 2=0.01%, 10=0.02%, 20=97.41%, 50=2.48%, 100=0.08%
  lat (usec)   : 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.01%
  cpu          : usr=6.58%, sys=36.71%, ctx=3517121, majf=0, minf=7
  IO depths    : 1=103.5%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwt: total=0,3517118,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=229MiB/s (240MB/s), 229MiB/s-229MiB/s (240MB/s-240MB/s), io=13.4GiB (14.4GB), run=60001-60001msec

Disk stats (read/write):
  nvme10n1: ios=60/3639178, merge=0/0, ticks=8/36190, in_queue=34354, util=55.34%
```

【**2. Random Write**】

```
sudo fio --name=random_write_latency_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=1 --rw=randwrite --group_reporting

random_write_latency_test: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [w(1)][100.0%][r=0KiB/s,w=211MiB/s][r=0,w=53.0k IOPS][eta 00m:00s]
write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=108503: Wed Jul 11 21:06:25 2018
  write: IOPS=53.2k, BW=208MiB/s (218MB/s)(12.2GiB/60001msec)
    slat (nsec): min=2685, max=46746, avg=3780.57, stdev=397.35
    clat (nsec): min=582, max=7269.0k, avg=14212.36, stdev=12824.43
     lat (usec): min=12, max=7273, avg=18.05, stdev=12.83
    clat percentiles (nsec):
     |  1.00th=[11328],  5.00th=[11456], 10.00th=[11712], 20.00th=[11840],
     | 30.00th=[12096], 40.00th=[12352], 50.00th=[12736], 60.00th=[13248],
     | 70.00th=[13760], 80.00th=[14272], 90.00th=[16512], 95.00th=[21888],
     | 99.00th=[42240], 99.50th=[47360], 99.90th=[62720], 99.95th=[74240],
     | 99.99th=[95744]
   bw (  KiB/s): min=199911, max=236592, per=100.00%, avg=214051.20, stdev=6703.14, samples=120
   iops        : min=49977, max=59148, avg=53512.45, stdev=1675.88, samples=120
  lat (nsec)   : 750=0.01%, 1000=0.01%
  lat (usec)   : 2=0.01%, 10=0.02%, 20=93.72%, 50=5.92%, 100=0.33%
  lat (usec)   : 250=0.01%, 500=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.01%
  cpu          : usr=7.08%, sys=34.24%, ctx=3193805, majf=0, minf=6
  IO depths    : 1=103.7%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwt: total=0,3193795,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=208MiB/s (218MB/s), 208MiB/s-208MiB/s (218MB/s-218MB/s), io=12.2GiB (13.1GB), run=60001-60001msec

Disk stats (read/write):
  nvme10n1: ios=60/3310392, merge=0/0, ticks=3/37140, in_queue=35308, util=56.88%
```



【**3. Sequential Read**】

```
sudo fio --name=sequential_read_latency_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=1 --rw=read --group_reporting

sequential_read_latency_test: (g=0): rw=read, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [R(1)][100.0%][r=254MiB/s,w=0KiB/s][r=65.0k,w=0 IOPS][eta 00m:00s]
write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=78515: Wed Jul 11 20:56:00 2018
   read: IOPS=64.2k, BW=251MiB/s (263MB/s)(14.7GiB/60001msec)
    slat (nsec): min=2447, max=35032, avg=3326.52, stdev=374.82
    clat (nsec): min=729, max=3531.5k, avg=11716.87, stdev=12612.79
     lat (usec): min=10, max=3535, avg=15.10, stdev=12.62
    clat percentiles (nsec):
     |  1.00th=[10688],  5.00th=[10944], 10.00th=[10944], 20.00th=[11200],
     | 30.00th=[11328], 40.00th=[11328], 50.00th=[11456], 60.00th=[11584],
     | 70.00th=[11712], 80.00th=[11840], 90.00th=[12224], 95.00th=[12736],
     | 99.00th=[13888], 99.50th=[14784], 99.90th=[30336], 99.95th=[42240],
     | 99.99th=[77312]
   bw (  KiB/s): min=237064, max=267991, per=100.00%, avg=257842.35, stdev=5278.64, samples=120
   iops        : min=59266, max=66997, avg=64460.29, stdev=1319.57, samples=120
  lat (nsec)   : 750=0.01%, 1000=0.01%
  lat (usec)   : 2=0.01%, 4=0.01%, 10=0.03%, 20=99.81%, 50=0.15%
  lat (usec)   : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%
  cpu          : usr=6.70%, sys=37.92%, ctx=3850007, majf=0, minf=6
  IO depths    : 1=103.2%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwt: total=3850011,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=251MiB/s (263MB/s), 251MiB/s-251MiB/s (263MB/s-263MB/s), io=14.7GiB (15.8GB), run=60001-60001msec
```



【**4. Random Read**】

```
sudo fio --name=random_read_latency_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=1 --rw=randread --group_reporting

random_read_latency_test: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [r(1)][100.0%][r=39.7MiB/s,w=0KiB/s][r=10.2k,w=0 IOPS][eta 00m:00s]
write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=67152: Wed Jul 11 20:51:34 2018
   read: IOPS=10.0k, BW=39.2MiB/s (41.1MB/s)(2352MiB/60001msec)
    slat (nsec): min=2584, max=28712, avg=3687.75, stdev=435.50
    clat (usec): min=39, max=2850, avg=95.15, stdev=69.53
     lat (usec): min=42, max=2854, avg=98.90, stdev=69.53
    clat percentiles (usec):
     |  1.00th=[   74],  5.00th=[   79], 10.00th=[   82], 20.00th=[   85],
     | 30.00th=[   85], 40.00th=[   86], 50.00th=[   89], 60.00th=[   99],
     | 70.00th=[  101], 80.00th=[  102], 90.00th=[  102], 95.00th=[  103],
     | 99.00th=[  120], 99.50th=[  210], 99.90th=[ 1450], 99.95th=[ 2057],
     | 99.99th=[ 2507]
   bw (  KiB/s): min=38933, max=41411, per=100.00%, avg=40348.50, stdev=447.97, samples=120
   iops        : min= 9733, max=10352, avg=10086.79, stdev=111.98, samples=120
  lat (usec)   : 50=0.01%, 100=64.62%, 250=34.96%, 500=0.20%, 750=0.04%
  lat (usec)   : 1000=0.04%
  lat (msec)   : 2=0.08%, 4=0.05%
  cpu          : usr=1.39%, sys=6.47%, ctx=602015, majf=0, minf=5
  IO depths    : 1=103.3%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwt: total=602009,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=39.2MiB/s (41.1MB/s), 39.2MiB/s-39.2MiB/s (41.1MB/s-41.1MB/s), io=2352MiB (2466MB), run=60001-60001msec
```

