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

|                  | BandWidth  | IOPS | Latency(usec)                                             |
| ---------------- | ---------- | ---- | --------------------------------------------------------- |
| Sequential Write | 1936.3MB/s | 288K | 2=0.01%, 4=0.01%, 【10=90.14%】, 20=9.33%, 50=0.52%       |
| Random Write     | 1937.7MB/s | 268K | 2=0.01%, 4=0.01%, 【10=93.16%】, 20=6.52%, 50=0.31%       |
| Sequential Read  | 2654.8MB/s | 296K | 2=0.01%, 4=0.01%, 【10=97.95%】, 20=2.01%, 50=0.01%       |
| Random Read      | 2599.9MB/s | 290K | 50=0.01%, 【100=97.46%】, 250=2.53%, 500=0.01%, 750=0.01% |

### 2.2 EXT4

|                  | BandWidth  | Latency(usec)                                           |
| ---------------- | ---------- | ------------------------------------------------------- |
| Sequential Write | 1934.3MB/s | 2=0.01%, 4=0.01%, 10=0.01%, 【20=99.18%】, 50=0.80%     |
| Random Write     | 1938.5MB/s | 2=0.01%, 4=0.01%, 10=0.01%, 【20=97.83%】, 50=2.11%     |
| Sequential Read  | 2668.9MB/s | 2=0.01%, 4=0.01%, 【10=97.69%】, 20=2.24%, 50=0.02%     |
| Random Read      | 2627.3MB/s | 4=0.01%, 50=0.01%, 【100=98.35%】, 250=1.63%, 500=0.01% |



**IOPS在EXT4上比较特殊，根据测试文件（100G）之前是如何生成的，会有不同的结果，如下：**

| IOPS             | 文件之前不存在 | 文件提前用dd生成（bs=128M，count=1000） |
| ---------------- | -------------- | --------------------------------------- |
| Sequential Write | 101K           | 227K                                    |
| Random Write     | 60K            | 216K                                    |
| Sequential Read  | 277K           | 269K                                    |
| Random Read      | 270K           | 265K                                    |



**结果比较：**

1. 无论裸盘还是EXT4，Write BandWidth整体比Read BandWidth少
2. 无论裸盘还是EXT4，Random Read的Latency相比其他Latency高很多
3. 在ETX4上，如果文件之前不存在，则Write IOPS相比裸盘低很多，通过顺序写先生成文件后，IOPS基本和裸盘一样



## 3. 测试详情

测试裸盘和在EXT4上的方法，除了参数filename不一样外，其他均一样，所以以测试裸盘为例，列出测试详情

**【呃。。这博客页面貌似支持不了一行太长的字符（我也懒得换行了），命令已经伸出黑框外了。。。需要拷贝命令的话直接用鼠标点选，那样会显示出所有信息】**

### 3.1 BandWidth

测试BandWidth时，bs和iodepth需要设置大一些才可以跑满BandWidth

bs=8m，iodepth=1024

【**1. Sequential Write**】

```
sudo fio --name=sequential_write_bandwidth_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=8M --iodepth=1024 --rw=write --group_reporting

sequential_write_bandwidth_test: (g=0): rw=write, bs=8M-8M/8M-8M/8M-8M, ioengine=libaio, iodepth=1024
fio-2.2.8
Starting 1 process
Jobs: 1 (f=0): [W(1)] [100.0% done] [0KB/9856MB/0KB /s] [0/1232/0 iops] [eta 00m:00s]
sequential_write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=47939: Thu Jul 12 19:08:10 2018
  write: io=116320MB, bw=1936.3MB/s, iops=225, runt= 60074msec
    slat (usec): min=430, max=22550, avg=4421.26, stdev=2450.11
    clat (msec): min=70, max=4722, avg=4228.70, stdev=906.55
     lat (msec): min=75, max=4730, avg=4233.12, stdev=906.56
    clat percentiles (msec):
     |  1.00th=[  400],  5.00th=[ 1680], 10.00th=[ 3294], 20.00th=[ 4490],
     | 30.00th=[ 4490], 40.00th=[ 4555], 50.00th=[ 4555], 60.00th=[ 4555],
     | 70.00th=[ 4555], 80.00th=[ 4555], 90.00th=[ 4555], 95.00th=[ 4621],
     | 99.00th=[ 4621], 99.50th=[ 4686], 99.90th=[ 4686], 99.95th=[ 4686],
     | 99.99th=[ 4752]
    bw (MB  /s): min=    1, max= 2176, per=91.37%, avg=1769.23, stdev=256.70
    lat (msec) : 100=0.10%, 250=0.51%, 500=0.80%, 750=0.84%, 1000=0.83%
    lat (msec) : 2000=3.35%, >=2000=101.14%
  cpu          : usr=13.16%, sys=10.68%, ctx=79385, majf=0, minf=31
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.2%, 32=0.5%, >=64=106.6%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=0/w=13517/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1024

Run status group 0 (all jobs):
  WRITE: io=116320MB, aggrb=1936.3MB/s, minb=1936.3MB/s, maxb=1936.3MB/s, mint=60074msec, maxt=60074msec

Disk stats (read/write):
  nvme10n1: ios=70/927000, merge=0/0, ticks=627/75752472, in_queue=75795598, util=69.63%
```

【**2. Random Write**】

```
sudo fio --name=random_write_bandwidth_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=8M --iodepth=1024 --rw=randwrite --group_reporting

random_write_bandwidth_test: (g=0): rw=randwrite, bs=8M-8M/8M-8M/8M-8M, ioengine=libaio, iodepth=1024
fio-2.2.8
Starting 1 process
Jobs: 1 (f=0): [w(1)] [100.0% done] [0KB/0KB/0KB /s] [0/0/0 iops] [eta 00m:00s]
random_write_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=24405: Thu Jul 12 19:05:04 2018
  write: io=116416MB, bw=1937.7MB/s, iops=225, runt= 60082msec
    slat (usec): min=522, max=23583, avg=4419.65, stdev=2279.16
    clat (msec): min=77, max=4729, avg=4229.20, stdev=907.68
     lat (msec): min=81, max=4738, avg=4233.62, stdev=907.69
    clat percentiles (msec):
     |  1.00th=[  392],  5.00th=[ 1713], 10.00th=[ 3294], 20.00th=[ 4490],
     | 30.00th=[ 4490], 40.00th=[ 4555], 50.00th=[ 4555], 60.00th=[ 4555],
     | 70.00th=[ 4555], 80.00th=[ 4555], 90.00th=[ 4555], 95.00th=[ 4621],
     | 99.00th=[ 4621], 99.50th=[ 4686], 99.90th=[ 4686], 99.95th=[ 4686],
     | 99.99th=[ 4752]
    bw (MB  /s): min=    1, max= 2351, per=91.39%, avg=1770.85, stdev=262.43
    lat (msec) : 100=0.07%, 250=0.44%, 500=0.92%, 750=0.81%, 1000=0.86%
    lat (msec) : 2000=3.24%, >=2000=101.21%
  cpu          : usr=12.96%, sys=11.22%, ctx=81730, majf=0, minf=32
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.2%, 32=0.5%, >=64=106.6%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=0/w=13529/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1024

Run status group 0 (all jobs):
  WRITE: io=116416MB, aggrb=1937.7MB/s, minb=1937.7MB/s, maxb=1937.7MB/s, mint=60082msec, maxt=60082msec

Disk stats (read/write):
  nvme10n1: ios=69/930688, merge=0/0, ticks=517/74473274, in_queue=74510317, util=64.79%
```



【**3. Sequential Read**】

```
sudo fio --name=sequential_read_bandwidth_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=8M --iodepth=1024 --rw=read --group_reporting

sequential_read_bandwidth_test: (g=0): rw=read, bs=8M-8M/8M-8M/8M-8M, ioengine=libaio, iodepth=1024
fio-2.2.8
Starting 1 process
Jobs: 1 (f=0): [R(1)] [100.0% done] [10392MB/0KB/0KB /s] [1299/0/0 iops] [eta 00m:00s]
sequential_read_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=68933: Thu Jul 12 19:10:54 2018
  read : io=159416MB, bw=2654.8MB/s, iops=314, runt= 60049msec
    slat (usec): min=391, max=97353, avg=4109.11, stdev=8772.17
    clat (msec): min=48, max=22006, avg=3636.76, stdev=2717.89
     lat (msec): min=51, max=22008, avg=3640.87, stdev=2720.99
    clat percentiles (msec):
     |  1.00th=[  412],  5.00th=[ 1614], 10.00th=[ 3097], 20.00th=[ 3195],
     | 30.00th=[ 3228], 40.00th=[ 3228], 50.00th=[ 3261], 60.00th=[ 3261],
     | 70.00th=[ 3294], 80.00th=[ 3294], 90.00th=[ 3359], 95.00th=[ 3523],
     | 99.00th=[16712], 99.50th=[16712], 99.90th=[16712], 99.95th=[16712],
     | 99.99th=[16712]
    bw (MB  /s): min=    0, max= 3145, per=93.31%, avg=2477.22, stdev=378.15
    lat (msec) : 50=0.01%, 100=0.17%, 250=0.48%, 500=0.93%, 750=0.86%
    lat (msec) : 1000=0.71%, 2000=3.41%, >=2000=98.85%
  cpu          : usr=0.07%, sys=53.72%, ctx=107871, majf=0, minf=2016960
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.2%, 32=0.3%, >=64=104.8%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=18904/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1024

Run status group 0 (all jobs):
   READ: io=159416MB, aggrb=2654.8MB/s, minb=2654.8MB/s, maxb=2654.8MB/s, mint=60049msec, maxt=60049msec

Disk stats (read/write):
  nvme10n1: ios=1272565/0, merge=0/0, ticks=73750196/0, in_queue=73800747, util=95.30%
```



【**4. Random Read**】

```
sudo fio --name=random_read_bandwidth_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=8M --iodepth=1024 --rw=randread --group_reporting

random_read_bandwidth_test: (g=0): rw=randread, bs=8M-8M/8M-8M/8M-8M, ioengine=libaio, iodepth=1024
fio-2.2.8
Starting 1 process
Jobs: 1 (f=0): [r(1)] [100.0% done] [10376MB/0KB/0KB /s] [1297/0/0 iops] [eta 00m:00s]
random_read_bandwidth_test: (groupid=0, jobs=1): err= 0: pid=85475: Thu Jul 12 19:13:04 2018
  read : io=156280MB, bw=2599.9MB/s, iops=307, runt= 60112msec
    slat (usec): min=378, max=97785, avg=3911.97, stdev=7576.38
    clat (msec): min=67, max=16526, avg=3535.58, stdev=2038.26
     lat (msec): min=67, max=16528, avg=3539.49, stdev=2040.38
    clat percentiles (msec):
     |  1.00th=[  375],  5.00th=[ 1631], 10.00th=[ 3163], 20.00th=[ 3294],
     | 30.00th=[ 3294], 40.00th=[ 3294], 50.00th=[ 3326], 60.00th=[ 3326],
     | 70.00th=[ 3326], 80.00th=[ 3359], 90.00th=[ 3392], 95.00th=[ 3458],
     | 99.00th=[15533], 99.50th=[15664], 99.90th=[16450], 99.95th=[16450],
     | 99.99th=[16581]
    bw (MB  /s): min=    0, max= 3152, per=93.10%, avg=2420.39, stdev=364.14
    lat (msec) : 100=0.07%, 250=0.52%, 500=0.89%, 750=0.69%, 1000=0.95%
    lat (msec) : 2000=3.34%, >=2000=99.06%
  cpu          : usr=0.08%, sys=44.31%, ctx=103207, majf=0, minf=2016449
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.2%, 32=0.3%, >=64=104.9%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=18512/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1024

Run status group 0 (all jobs):
   READ: io=156280MB, aggrb=2599.9MB/s, minb=2599.9MB/s, maxb=2599.9MB/s, mint=60112msec, maxt=60112msec

Disk stats (read/write):
  nvme10n1: ios=1246153/0, merge=0/0, ticks=74845380/0, in_queue=74908662, util=96.46%
```

### 

### 3.2 IOPS

测试IOPS时，bs不能大，否则可能BandWidth先达到瓶颈导致IOPS测试不准，另外iodepth需要设置大一些。

bs=4k，iodepth=4096

【**1. Sequential Write**】

```
sudo fio --name=sequential_write_iops_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=4096 --rw=write --group_reporting

sequential_write_iops_test: (g=0): rw=write, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=4096
fio-2.2.8
Starting 1 process
Jobs: 1 (f=1): [W(1)] [100.0% done] [0KB/970.2MB/0KB /s] [0/248K/0 iops] [eta 00m:00s]
sequential_write_iops_test: (groupid=0, jobs=1): err= 0: pid=113737: Thu Jul 12 19:16:45 2018
  write: io=67429MB, bw=1123.9MB/s, iops=287623, runt= 60001msec
    slat (usec): min=1, max=4107, avg= 2.60, stdev= 4.93
    clat (usec): min=50, max=21711, avg=14235.74, stdev=772.57
     lat (usec): min=53, max=21714, avg=14238.39, stdev=772.71
    clat percentiles (usec):
     |  1.00th=[13632],  5.00th=[13760], 10.00th=[13760], 20.00th=[13760],
     | 30.00th=[13760], 40.00th=[13888], 50.00th=[13888], 60.00th=[14016],
     | 70.00th=[14272], 80.00th=[14784], 90.00th=[15040], 95.00th=[15424],
     | 99.00th=[17536], 99.50th=[18560], 99.90th=[19840], 99.95th=[20096],
     | 99.99th=[21120]
    bw (MB  /s): min=    0, max= 1158, per=99.23%, avg=1115.11, stdev=109.80
    lat (usec) : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
    lat (msec) : 2=0.01%, 4=0.01%, 10=0.01%, 20=99.92%, 50=0.09%
  cpu          : usr=27.65%, sys=75.50%, ctx=292, majf=0, minf=34
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=103.4%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=0/w=17257686/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
  WRITE: io=67429MB, aggrb=1123.9MB/s, minb=1123.9MB/s, maxb=1123.9MB/s, mint=60001msec, maxt=60001msec

Disk stats (read/write):
  nvme10n1: ios=30/17824369, merge=0/0, ticks=9/823892, in_queue=822674, util=98.96%
```

【**2. Random Write**】

```
sudo fio --name=random_write_iops_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=4096 --rw=randwrite --group_reporting

random_write_iops_test: (g=0): rw=randwrite, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=4096
fio-2.2.8
Starting 1 process
Jobs: 1 (f=1): [w(1)] [100.0% done] [0KB/1075MB/0KB /s] [0/275K/0 iops] [eta 00m:00s]
random_write_iops_test: (groupid=0, jobs=1): err= 0: pid=128746: Thu Jul 12 19:18:48 2018
  write: io=62898MB, bw=1048.3MB/s, iops=268292, runt= 60001msec
    slat (usec): min=1, max=3842, avg= 2.63, stdev= 4.78
    clat (usec): min=53, max=23495, avg=15261.50, stdev=812.29
     lat (usec): min=55, max=23497, avg=15264.17, stdev=812.42
    clat percentiles (usec):
     |  1.00th=[14528],  5.00th=[14528], 10.00th=[14656], 20.00th=[14784],
     | 30.00th=[14784], 40.00th=[14784], 50.00th=[14912], 60.00th=[15040],
     | 70.00th=[15296], 80.00th=[16064], 90.00th=[16320], 95.00th=[16512],
     | 99.00th=[18304], 99.50th=[19584], 99.90th=[21376], 99.95th=[22144],
     | 99.99th=[22912]
    bw (MB  /s): min=    0, max= 1091, per=99.21%, avg=1039.96, stdev=102.54
    lat (usec) : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
    lat (msec) : 2=0.01%, 4=0.01%, 10=0.01%, 20=99.67%, 50=0.34%
  cpu          : usr=32.09%, sys=71.09%, ctx=215, majf=0, minf=34
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=103.5%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=0/w=16097801/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
  WRITE: io=62898MB, aggrb=1048.3MB/s, minb=1048.3MB/s, maxb=1048.3MB/s, mint=60001msec, maxt=60001msec

Disk stats (read/write):
  nvme10n1: ios=25/16633027, merge=0/0, ticks=10/806595, in_queue=805081, util=99.05%
```



【**3. Sequential Read**】

```
sudo fio --name=sequential_read_iops_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=4096 --rw=read --group_reporting

sequential_read_iops_test: (g=0): rw=read, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=4096
fio-2.2.8
Starting 1 process
Jobs: 1 (f=1): [R(1)] [100.0% done] [1148MB/0KB/0KB /s] [294K/0/0 iops] [eta 00m:00s]
sequential_read_iops_test: (groupid=0, jobs=1): err= 0: pid=9785: Thu Jul 12 19:20:25 2018
  read : io=69357MB, bw=1155.1MB/s, iops=295848, runt= 60001msec
    slat (usec): min=1, max=590, avg= 2.59, stdev= 1.24
    clat (usec): min=183, max=18868, avg=13840.05, stdev=200.10
     lat (usec): min=186, max=18871, avg=13842.70, stdev=200.12
    clat percentiles (usec):
     |  1.00th=[13632],  5.00th=[13632], 10.00th=[13632], 20.00th=[13760],
     | 30.00th=[13760], 40.00th=[13760], 50.00th=[13760], 60.00th=[13888],
     | 70.00th=[13888], 80.00th=[13888], 90.00th=[14016], 95.00th=[14144],
     | 99.00th=[14400], 99.50th=[14528], 99.90th=[15296], 99.95th=[15680],
     | 99.99th=[16192]
    bw (MB  /s): min=    0, max= 1168, per=99.15%, avg=1146.10, stdev=105.75
    lat (usec) : 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
    lat (msec) : 2=0.01%, 4=0.01%, 10=0.01%, 20=100.01%
  cpu          : usr=21.55%, sys=81.89%, ctx=145, majf=0, minf=555
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=103.4%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=17751206/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
   READ: io=69357MB, aggrb=1155.1MB/s, minb=1155.1MB/s, maxb=1155.1MB/s, mint=60001msec, maxt=60001msec

Disk stats (read/write):
  nvme10n1: ios=18329625/0, merge=0/0, ticks=2635338/0, in_queue=2661197, util=99.98%
```



【**4. Random Read**】

```
sudo fio --name=random_read_iops_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=4096 --rw=randread --group_reporting

random_read_iops_test: (g=0): rw=randread, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=4096
fio-2.2.8
Starting 1 process
Jobs: 1 (f=1): [r(1)] [100.0% done] [1119MB/0KB/0KB /s] [287K/0/0 iops] [eta 00m:00s]
random_read_iops_test: (groupid=0, jobs=1): err= 0: pid=21410: Thu Jul 12 19:22:01 2018
  read : io=67788MB, bw=1129.8MB/s, iops=289153, runt= 60001msec
    slat (usec): min=1, max=572, avg= 2.48, stdev= 1.13
    clat (usec): min=170, max=17820, avg=14160.40, stdev=273.83
     lat (usec): min=173, max=17822, avg=14162.94, stdev=273.85
    clat percentiles (usec):
     |  1.00th=[13888],  5.00th=[13888], 10.00th=[13888], 20.00th=[14016],
     | 30.00th=[14016], 40.00th=[14016], 50.00th=[14144], 60.00th=[14144],
     | 70.00th=[14272], 80.00th=[14400], 90.00th=[14528], 95.00th=[14656],
     | 99.00th=[14912], 99.50th=[15040], 99.90th=[15680], 99.95th=[16320],
     | 99.99th=[16512]
    bw (MB  /s): min=    0, max= 1151, per=99.15%, avg=1120.19, stdev=104.25
    lat (usec) : 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
    lat (msec) : 2=0.01%, 4=0.01%, 10=0.01%, 20=100.01%
  cpu          : usr=26.87%, sys=76.58%, ctx=153, majf=0, minf=555
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=103.4%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=17349528/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=4096

Run status group 0 (all jobs):
   READ: io=67788MB, aggrb=1129.8MB/s, minb=1129.8MB/s, maxb=1129.8MB/s, mint=60001msec, maxt=60001msec

Disk stats (read/write):
  nvme10n1: ios=17914488/0, merge=0/0, ticks=2225497/0, in_queue=2241388, util=99.94%
```

### 

### 3.3 Latency

测试Latency时，bs和iodepth都需要设置小一点，否则如果bandwidth或IPOS先到瓶颈的话，会导致latency不准确

bs=4k，iodepth=1

【**1. Sequential Write**】

```
sudo fio --name=sequential_write_latency_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=1 --rw=write --group_reporting

sequential_write_latency_test: (g=0): rw=write, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=1
fio-2.2.8
Starting 1 process
Jobs: 1 (f=1): [W(1)] [100.0% done] [0KB/376.6MB/0KB /s] [0/96.4K/0 iops] [eta 00m:00s]
sequential_write_latency_test: (groupid=0, jobs=1): err= 0: pid=33594: Thu Jul 12 19:23:40 2018
  write: io=22463MB, bw=383367KB/s, iops=95841, runt= 60001msec
    slat (usec): min=1, max=420, avg= 1.55, stdev= 0.64
    clat (usec): min=0, max=7649, avg= 8.42, stdev=12.73
     lat (usec): min=7, max=7651, avg=10.00, stdev=12.74
    clat percentiles (usec):
     |  1.00th=[    7],  5.00th=[    7], 10.00th=[    7], 20.00th=[    7],
     | 30.00th=[    7], 40.00th=[    8], 50.00th=[    8], 60.00th=[    8],
     | 70.00th=[    8], 80.00th=[    8], 90.00th=[    9], 95.00th=[   16],
     | 99.00th=[   17], 99.50th=[   20], 99.90th=[   25], 99.95th=[   26],
     | 99.99th=[   31]
    bw (KB  /s): min=    2, max=389968, per=99.17%, avg=380186.35, stdev=35315.31
    lat (usec) : 2=0.01%, 4=0.01%, 10=90.14%, 20=9.33%, 50=0.52%
    lat (usec) : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
    lat (msec) : 2=0.01%, 10=0.01%
  cpu          : usr=18.68%, sys=18.95%, ctx=5943202, majf=0, minf=31
  IO depths    : 1=103.4%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=5750603/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: io=22463MB, aggrb=383367KB/s, minb=383367KB/s, maxb=383367KB/s, mint=60001msec, maxt=60001msec

Disk stats (read/write):
  nvme10n1: ios=30/5935264, merge=0/0, ticks=5/42293, in_queue=41933, util=67.64%
```

【**2. Random Write**】

```
sudo fio --name=random_write_latency_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=1 --rw=randwrite --group_reporting

random_write_latency_test: (g=0): rw=randwrite, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=1
fio-2.2.8
Starting 1 process
Jobs: 1 (f=1): [w(1)] [100.0% done] [0KB/369.6MB/0KB /s] [0/94.5K/0 iops] [eta 00m:00s]
random_write_latency_test: (groupid=0, jobs=1): err= 0: pid=48951: Thu Jul 12 19:25:48 2018
  write: io=22293MB, bw=380456KB/s, iops=95113, runt= 60001msec
    slat (usec): min=1, max=516, avg= 1.62, stdev= 0.65
    clat (usec): min=0, max=7559, avg= 8.24, stdev=12.71
     lat (usec): min=7, max=7561, avg= 9.90, stdev=12.72
    clat percentiles (usec):
     |  1.00th=[    7],  5.00th=[    7], 10.00th=[    7], 20.00th=[    7],
     | 30.00th=[    8], 40.00th=[    8], 50.00th=[    8], 60.00th=[    8],
     | 70.00th=[    8], 80.00th=[    8], 90.00th=[    9], 95.00th=[   11],
     | 99.00th=[   17], 99.50th=[   18], 99.90th=[   24], 99.95th=[   27],
     | 99.99th=[   32]
    bw (KB  /s): min=    2, max=387160, per=99.18%, avg=377316.68, stdev=35008.78
    lat (usec) : 2=0.01%, 4=0.01%, 10=93.16%, 20=6.52%, 50=0.31%
    lat (usec) : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
    lat (msec) : 10=0.01%
  cpu          : usr=20.88%, sys=19.36%, ctx=5898649, majf=0, minf=30
  IO depths    : 1=103.4%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=5706933/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: io=22293MB, aggrb=380455KB/s, minb=380455KB/s, maxb=380455KB/s, mint=60001msec, maxt=60001msec

Disk stats (read/write):
  nvme10n1: ios=26/5890770, merge=0/0, ticks=1/40884, in_queue=40532, util=65.38%
```



【**3. Sequential Read**】

```
sudo fio --name=sequential_read_latency_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=1 --rw=read --group_reporting

sequential_read_latency_test: (g=0): rw=read, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=1
fio-2.2.8
Starting 1 process
Jobs: 1 (f=1): [R(1)] [100.0% done] [411.4MB/0KB/0KB /s] [105K/0/0 iops] [eta 00m:00s]
sequential_read_latency_test: (groupid=0, jobs=1): err= 0: pid=65274: Thu Jul 12 19:28:02 2018
  read : io=24687MB, bw=421319KB/s, iops=105329, runt= 60001msec
    slat (usec): min=1, max=456, avg= 1.51, stdev= 0.62
    clat (usec): min=0, max=3031, avg= 7.53, stdev= 9.75
     lat (usec): min=7, max=3032, avg= 9.08, stdev= 9.77
    clat percentiles (usec):
     |  1.00th=[    7],  5.00th=[    7], 10.00th=[    7], 20.00th=[    7],
     | 30.00th=[    7], 40.00th=[    7], 50.00th=[    7], 60.00th=[    7],
     | 70.00th=[    8], 80.00th=[    8], 90.00th=[    8], 95.00th=[    9],
     | 99.00th=[   10], 99.50th=[   11], 99.90th=[   12], 99.95th=[   14],
     | 99.99th=[  278]
    bw (KB  /s): min=    2, max=428168, per=99.17%, avg=417814.08, stdev=38680.41
    lat (usec) : 2=0.01%, 4=0.01%, 10=97.95%, 20=2.01%, 50=0.01%
    lat (usec) : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
    lat (msec) : 2=0.01%, 4=0.01%
  cpu          : usr=19.67%, sys=20.98%, ctx=6528967, majf=0, minf=34
  IO depths    : 1=103.3%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=6319891/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: io=24687MB, aggrb=421319KB/s, minb=421319KB/s, maxb=421319KB/s, mint=60001msec, maxt=60001msec

Disk stats (read/write):
  nvme10n1: ios=6520893/0, merge=0/0, ticks=40553/0, in_queue=40187, util=64.82%
```



【**4. Random Read**】

```
sudo fio --name=random_read_latency_test --filename=/dev/nvme10n1 --filesize=100G --time_based --ramp_time=2s --runtime=1m --ioengine=libaio --direct=1 --verify=0 --randrepeat=0 --bs=4k --iodepth=1 --rw=randread --group_reporting

random_read_latency_test: (g=0): rw=randread, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=1
fio-2.2.8
Starting 1 process
Jobs: 1 (f=1): [r(1)] [100.0% done] [44736KB/0KB/0KB /s] [11.2K/0/0 iops] [eta 00m:00s]
random_read_latency_test: (groupid=0, jobs=1): err= 0: pid=78159: Thu Jul 12 19:29:49 2018
  read : io=2581.1MB, bw=44064KB/s, iops=11016, runt= 60001msec
    slat (usec): min=1, max=1031, avg= 2.17, stdev= 1.57
    clat (usec): min=36, max=2578, avg=87.76, stdev=13.31
     lat (usec): min=38, max=2579, avg=89.99, stdev=13.43
    clat percentiles (usec):
     |  1.00th=[   69],  5.00th=[   74], 10.00th=[   77], 20.00th=[   80],
     | 30.00th=[   81], 40.00th=[   82], 50.00th=[   83], 60.00th=[   94],
     | 70.00th=[   97], 80.00th=[   98], 90.00th=[   98], 95.00th=[   99],
     | 99.00th=[  100], 99.50th=[  100], 99.90th=[  102], 99.95th=[  108],
     | 99.99th=[  290]
    bw (KB  /s): min=    2, max=44864, per=99.17%, avg=43697.82, stdev=4064.10
    lat (usec) : 50=0.01%, 100=97.46%, 250=2.53%, 500=0.01%, 750=0.01%
    lat (usec) : 1000=0.01%
    lat (msec) : 2=0.01%, 4=0.01%
  cpu          : usr=2.92%, sys=3.21%, ctx=683299, majf=0, minf=33
  IO depths    : 1=103.4%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=660978/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: io=2581.1MB, aggrb=44064KB/s, minb=44064KB/s, maxb=44064KB/s, mint=60001msec, maxt=60001msec

Disk stats (read/write):
  nvme10n1: ios=682327/0, merge=0/0, ticks=58895/0, in_queue=58839, util=94.91%
```

