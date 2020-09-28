编译构建：





构建主从流复制调试环境： 

1）在同一容器里面搭建主从postgresql服务；

mkdir /usr/local/pgsql/data5438

 mkdir /usr/local/pgsql/data5439

chown postgres:postgres /usr/local/pgsql/data543*

su - postgres

/usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data5438/

 vim postgresql.conf



listen_addresses = '*' # what IP address(es) to listen on;
port = 5438 # (change requires restart)
wal_level = replica # minimal, replica, or logical
synchronous_commit = on # synchronization level;
max_wal_senders = 10 # max number of walsender processes
wal_keep_segments = 1000 # in logfile segments, 16MB each; 0 disables
hot_standby = on # "off" disallows queries during recovery
log_destination = 'csvlog' # Valid values are combinations of
logging_collector = on # Enable capturing of stderr and csvlog
log_directory = 'log' # directory where log files are written,
log_min_duration_statement = 1000 # -1 is disabled, 0 logs all statements
log_checkpoints = on
log_connections = on
log_disconnections = on
log_duration = on
log_line_prefix = '%m %a %r %d %u %p %x' # special values:
log_statement = 'all' # none, ddl, mod, all
log_timezone = 'PRC'


vim /usr/local/pgsql/data5438/pg_hba.conf 

[postgres@bbaa2faf096e ~]$ cat /usr/local/pgsql/data5438/pg_hba.conf |grep -v "#"

local all all trust
host all all 127.0.0.1/32 trust
host all all ::1/128 trust
local replication all trust
host replication all 127.0.0.1/32 trust
host replication all ::1/128 trust

创建复制角色：
[postgres@bbaa2faf096e ~]$ /usr/local/pgsql/bin/psql -p 5438
psql (10.6)
Type "help" for help.

postgres=# create role repuser with replication login ;
CREATE ROLE
postgres=# \password repuser
Enter new password: 
Enter it again: 
/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data5438/ -l logfile start



/usr/local/pgsql/bin/pg_basebackup -h localhost -U repuser -p 5438 -D /usr/local/pgsql/data5439/



修改recovery.conf 文件：

vim /usr/local/pgsql/data5439/recovery.conf 
standby_mode = 'on'
primary_conninfo = 'host=localhost port=5438 user=repuser password=repuser'
recovery_target_timeline = 'latest'
chmod 0600 recovery.conf 

/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data5439/ -l logfile2 start

ps aux



2）在Master节点上打入一定量的数据：

/usr/local/pgsql/bin/pgbench -i -s 100 -h localhost -p 5438



3）查看主从服务相关进程；

[postgres@bbaa2faf096e ~]$ ps auxef
USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
postgres 16375 0.0 0.1 169792 13044 ? S 10:38 0:00 /usr/local/pgsql/bin/postgres -D /usr/local/pgsql/data5439 HOSTNAME=bba
postgres 16376 0.0 0.0 22624 884 ? Ss 10:38 0:00 \_ postgres: logger process 
postgres 16377 1.3 1.6 169888 136000 ? Ss 10:38 0:09 \_ postgres: startup process recovering 00000001000000000000004F 004
postgres 16378 0.0 1.1 169996 93772 ? Ss 10:38 0:00 \_ postgres: checkpointer process 
postgres 16379 0.1 1.6 169792 133632 ? Ss 10:38 0:00 \_ postgres: writer process 
postgres 16380 0.0 0.0 24740 748 ? Ss 10:38 0:00 \_ postgres: stats collector process 
postgres 16381 0.6 0.0 174508 2424 ? Ss 10:38 0:04 \_ postgres: wal receiver process streaming 0/4FD7A680 

postgres 16279 0.0 0.1 169792 13064 ? S 10:29 0:00 /usr/local/pgsql/bin/postgres -D /usr/local/pgsql/data5438 HOSTNAME=bba
postgres 16280 0.0 0.0 22624 892 ? Ss 10:29 0:00 \_ postgres: logger process 
postgres 16282 0.0 0.3 170104 24792 ? Ss 10:29 0:00 \_ postgres: checkpointer process 
postgres 16283 0.0 0.0 169792 2416 ? Ss 10:29 0:00 \_ postgres: writer process 
postgres 16284 0.1 0.0 169792 5324 ? Ss 10:29 0:01 \_ postgres: wal writer process 
postgres 16285 0.0 0.0 170208 2088 ? Ss 10:29 0:00 \_ postgres: autovacuum launcher process 
postgres 16286 0.0 0.0 24872 1084 ? Ss 10:29 0:00 \_ postgres: stats collector process 
postgres 16287 0.0 0.0 170084 1576 ? Ss 10:29 0:00 \_ postgres: bgworker: logical replication launcher 
postgres 16382 0.0 0.0 170612 2984 ? Ss 10:38 0:00 \_ postgres: wal sender process repuser ::1(48896) streaming 0/4FD7A68
4） gdb attach 调试主从进程，wal receiver 进程；



[postgres@bbaa2faf096e data5438]$ gdb 
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-114.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law. Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
(gdb) attach 16381      //attach到正确的进程；
Attaching to process 16381
Reading symbols from /usr/local/pgsql/bin/postgres...done.
Reading symbols from /lib64/libpthread.so.0...(no debugging symbols found)...done.
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
Loaded symbols for /lib64/libpthread.so.0
Reading symbols from /lib64/librt.so.1...(no debugging symbols found)...done.
Loaded symbols for /lib64/librt.so.1
Reading symbols from /lib64/libdl.so.2...(no debugging symbols found)...done.
Loaded symbols for /lib64/libdl.so.2
Reading symbols from /lib64/libm.so.6...(no debugging symbols found)...done.
Loaded symbols for /lib64/libm.so.6
Reading symbols from /lib64/libc.so.6...(no debugging symbols found)...done.
Loaded symbols for /lib64/libc.so.6
Reading symbols from /lib64/ld-linux-x86-64.so.2...(no debugging symbols found)...done.
Loaded symbols for /lib64/ld-linux-x86-64.so.2
Reading symbols from /lib64/libnss_files.so.2...(no debugging symbols found)...done.
Loaded symbols for /lib64/libnss_files.so.2
Reading symbols from /usr/local/pgsql/lib/libpqwalreceiver.so...done.
Loaded symbols for /usr/local/pgsql/lib/libpqwalreceiver.so
Reading symbols from /usr/local/pgsql/lib/libpq.so.5...done.
Loaded symbols for /usr/local/pgsql/lib/libpq.so.5
0x00007f1c16e0e463 in __epoll_wait_nocancel () from /lib64/libc.so.6
Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7_6.5.x86_64
(gdb) b WalReceiverMain     //设置断点
Breakpoint 1 at 0x7c1639: file walreceiver.c, line 197.
(gdb) i b                                //检查断点是否正确 
Num Type Disp Enb Address What
1 breakpoint keep y 0x00000000007c1639 in WalReceiverMain at walreceiver.c:197
(gdb) c                                  //继续执行，创造条件进入断点；
Continuing.



Program received signal SIGUSR1, User defined signal 1.
0x00007f1c16e0e463 in __epoll_wait_nocancel () from /lib64/libc.so.6
(gdb) bt                                //打印调用栈，调用顺序从下往上，返回源码查看分析调用过程；
#0 0x00007f1c16e0e463 in __epoll_wait_nocancel () from /lib64/libc.so.6
#1 0x00000000007f1775 in WaitEventSetWaitBlock (set=0x2754768, cur_timeout=100, occurred_events=0x7fff8684a370, nevents=1) at latch.c:1048
#2 0x00000000007f1650 in WaitEventSetWait (set=0x2754768, timeout=100, occurred_events=0x7fff8684a370, nevents=1, wait_event_info=83886091)
at latch.c:1000
#3 0x00000000007f0f60 in WaitLatchOrSocket (latch=0x7f1c16aaec34, wakeEvents=27, sock=3, timeout=100, wait_event_info=83886091) at latch.c:385
#4 0x00000000007c1f47 in WalReceiverMain () at walreceiver.c:501
#5 0x000000000052d76e in AuxiliaryProcessMain (argc=2, argv=0x7fff8684a9d0) at bootstrap.c:447
#6 0x000000000078f42e in StartChildProcess (type=WalReceiverProcess) at postmaster.c:5361
#7 0x000000000078f7fd in MaybeStartWalReceiver () at postmaster.c:5515
#8 0x000000000078f1cb in sigusr1_handler (postgres_signal_arg=10) at postmaster.c:5167
#9 <signal handler called>
#10 0x00007f1c16e04f53 in __select_nocancel () from /lib64/libc.so.6
#11 0x000000000078ae8d in ServerLoop () at postmaster.c:1719
#12 0x000000000078a638 in PostmasterMain (argc=3, argv=0x2753930) at postmaster.c:1363
#13 0x00000000006d2477 in main (argc=3, argv=0x2753930) at main.c:228
(gdb) c
Continuing.
