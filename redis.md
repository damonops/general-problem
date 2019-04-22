**现象**  
1. 线上某redis只能get值不能做更新操作，即set del等操作，具体错误信息如下：
MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk. Commands that may modify the data set are disabled. Please check Redis logs for details about the error。
2. redis Can't save in background: fork: Cannot allocate memory

**处理故障**
1. redis持久化到磁盘失败，有个选项参数是disabled，参数`stop-writes-on-bgsave-error yes`表示在redis刷新数据到磁盘失败的时候停止redis的写操作，故障处理原则：第一时间恢复，终端临时设置参数`config set stop-writes-on-bgsave-error no`，然后再修改配置文件，不用重启redis实例，再次set值OK；
2. 直接修改内核参数vm.overcommit_memory = 1

**分析**
1. 查看redis日志，Can’t save in background: fork: Cannot allocate memory ，很明确，内存不够了，为什么没收到告警呢？   

这里redis主机内存监控告警阈值设置有问题，当时主机内存还有将近2G可用，为什么还不够？还得从redis持久化原理说起。
redis持久化方式不做解析，我们用的rdb，一台主机有4个实例，每个实例占用内存2G(不包含碎片)，rdb文件大小在360M左右，主机内存16G(云主机实际是15.5G左右)，还跑一个java应用(占用内存4.2G)，codis-proxy消耗内存650M，所以应用总的消耗内存14G左右；redis在持久化数据到rdb文件的时候会fork主进程，异步回写数据，从而不使redis主进程阻塞，在回写的时候申请一块rdb文件大小的内存，持久化结束后覆盖原rdb文件，4个redis实例持久化需要1.5G内存，总内存就到了15.5，别忘了，操作系统消耗内存还没计算。
inux操作系统有个CommitLimit 限制系统应用使用的内存
```
grep -i commit /proc/meminfo
CommitLimit:     8610496 kB
Committed_AS:   15729928 kB
其中： CommitLimit，内存分配上限；Committed_AS，已分配内存
```
虚拟内存算法：
CommitLimit = 物理内存 * overcommit_ratio + swap  # overcommit_ratio内核参数，默认50%
当应用申请内存的时候，出现以下情况：
        应用程序要申请内存 + 系统已分配内存 > CommitLimit
内存不足，然后操作系统如何分配内存呢？需要overcommit_memory参数控制。
    
vm.overcommit_memory，该参数有三个值，分别0、1、2，操作系统默认0
- 0：请求分配虚拟内存大小和系统当前空闲物理内存作比价，请求内存大于物理内存，fork、malloc等调用失败
- 1：对内存资源不限制，避免fork失败，但malloc是先分配虚拟地址空间，然后通过异常陷入内核分配真正的物理内存，在内存不足的情况下，完全屏蔽了应用进程对系统内存状态的感知，即malloc总能成功，一旦内存不足，触发系统OOM，应用程序对此无法预测。
- 2：分配内存不超过CommitLimit上限，超过则错误，应用无法启动
> overcommit_ratio内核参数,只有在overcommit_memory=2时有效。

综上，最终我们调整redis的stop-writes-on-bgsave-error no,内存参数overcommit_memory=1,当然并不是最终解决，因为明显主机内存是不够的，清理redis数据？升级内存？迁移Java降低主机内存？见鬼去吧。  

附：通过总结，对于数据存储型应用，比如MySQL、redis、pipelinedb、postgresql、memcache、mongoDB等等，调整内核参数overcommit_memory=1，并考虑应用运行机制对内存监控设置不同的告警阈值，以避免OOM

OOM设置
```
echo 值 > /proc/$PID/oom_adj 
centos7.x /proc/$PID/oom_adj 废弃，使用/proc/$PID/oom_score_adj代替
```
“值”的范围 -16~15,值越大，越容易被系统OOM，有个特殊值-17，该进程服务禁止触发OOM机制
内存通过特定的算法给每个进程计算一个分数来决定kill哪个进程，每个进程的分数可以查看`/proc/$PID/oom_score`(或者**oom_score_adj**)
oom_score等于2的n次方，n为oom_adj值，得分越高越容易被kill.
内核参数`vm.panic_on_oom = 1`，禁止OOM机制

2.Redis的数据回写机制分同步和异步两种
- 同步回写即SAVE命令，主进程直接向磁盘回写数据。在数据大的情况下会导致系统假死很长时间，所以一般不是推荐的。
- 异步回写即BGSAVE命令，主进程fork后，复制自身并通过这个新的进程回写磁盘，回写结束后新进程自行关闭。由于这样做不需要主进程阻塞，系统不会假死，一般默认会采用这个方法。
个人感觉方法２采用fork主进程的方式很拙劣，但似乎是唯一的方法。内存中的热数据随时可能修改，要在磁盘上保存某个时间的内存镜像必须要冻结。冻结就会导致假死。fork一个新的进程之后等于复制了当时的一个内存镜像，这样主进程上就不需要冻结，只要子进程上操作就可以了。
在小内存的进程上做一个fork,不需要太多资源，但当这个进程的内存空间以Ｇ为单位时，fork就成为一件很恐怖的操作。何况在16G内存的主机上fork 14G内存的进程呢？肯定会报内存无法分配的。更可气的是，越是改动频繁的主机上fork也越频繁，fork操作本身的代价恐怕也不会比假死好多少。

