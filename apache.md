现象：Apache: No space left on device: Couldn’t create accept lock
分析：登录主机查看，系统磁盘仍有可用空间
搜索资料，涉及到系统的**active semaphores**

查看系统信号量：
```
ipcs -s
```
清理Apache占用信号量
```
ipcs -s |awk '/www-data/{print $2}' |xargs -n1 ipcrm -s
systemctl restart httpd  //一定要重启Apache
```

接下来，调整内核，增加系统进程信号量
```
kernel.msgmni = 65535 //系统中允许运行的最大message queue
kernel.sem = 500 6000000 500 1024 //centos7.5默认值：kernel.sem = 250    32000    32    128
sem四个参数含义：
第一列SEMMSL，表示每个信号集(set)中最大信号量数目
第二列SEMMNS，表示系统范围内的最大信号量总数目
第三列SEMOPM，表示每次信号操作，系统调用的最大信号量数目，此值应该与第一列(SEMMSL)相同
第四列SEMMNI，表示系统范围内的最大信号集(set)总数目
```
有关`kernel.sem`,参考[RedHat document](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/tuning_and_optimizing_red_hat_enterprise_linux_for_oracle_9i_and_10g_databases/sect-oracle_9i_and_10g_tuning_guide-setting_semaphores-setting_semaphore_parameters),红帽官方文档知识很全的。  

完整内核参数示例
```
net.ipv4.ip_forward = 0
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_retries2 = 5
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.conf.default.accept_source_route = 0
kernel.core_uses_pid = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.core.somaxconn = 65535
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_max_syn_backlog = 655360
net.ipv4.tcp_syn_retries = 1
fs.file-max = 1000000
fs.inotify.max_user_watches = 10000000
net.core.wmem_max = 327679
net.core.rmem_max = 327679
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_max_orphans = 65535
net.ipv4.tcp_keepalive_time = 30
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_wmem = 8192 436600 873200
net.ipv4.tcp_rmem = 32768 436600 873200
vm.swappiness = 0
vm.overcommit_memory = 1
net.ipv4.tcp_max_tw_buckets = 18000
kernel.sem = 500 6000000 500 1024
kernel.msgmni = 65535
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 68719476736  ##共享内存段最大尺寸，单位字节，这里64GB
kernel.shmmni = 4096         ##共享内存段的最大数量
kernel.shmall = 4294967296   ##系统一次可以使用的共享内存总量，单位page，
                             ##centos7.5默认page_size:4096，通过`getconf PAGESIZE/PAGE_SIZE`查看
```
