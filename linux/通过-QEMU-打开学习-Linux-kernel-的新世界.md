
最近面试了一位同学，在校期间对 QEMU 进行了二次开发，实现了设备建模、故障注入、信息提取功能。面试结束后感觉挺受启发，所以自己又多了解了一下 QEMU，发现可以基于 QEMU debug Linux kernel，打开了一个学习 kernel 的新世界。

## 1. 编译可 debug 的 Linux kernel

https://www.kernel.org/

下了最新版本 linux-5.13.12.tar.xz

#### 配置

通过 TUI 界面配置：
```
~/linux-5.13.12$ make menuconfig
```

开启 debug 相关的选项：
```
Kernel hacking  --->
  Compile-time checks and compiler options  --->
    [*] Compile the kernel with debug info
    [*]   Provide GDB scripts for kernel debugging
```

关闭内核地址空间布局随机：
```
Processor type and features  --->
  [ ]   Randomize the address of the kernel image (KASLR)
```

关闭签名检查：
```
-*- Cryptographic API  --->
  Certificates for signature checking  --->
    (debian/canonical-certs.pem) Additional X.509 keys for default system keyring
```

#### 编译

开始编译（很慢......）
```
~/linux-5.13.12$ make -j$(nproc)
```


## 2. 构建 initramfs

#### initramfs 是什么？

initramfs 是一种以 cpio 格式压缩的根文件系统，它通常和 Linux 内核文件 vmlinuz 一起被打包成 boot.img 作为启动镜像。

#### 为什么需要 initramfs？

内核启动的时候非常矛盾， boot loader 加载完内核文件 vmlinuz 后，内核紧接着需要挂载磁盘根文件系统，但如果此时内核没有相应驱动，无法识别磁盘，就需要先加载驱动，而驱动又位于 /lib/modules，得挂载根文件系统才能读取，这就陷入了一个两难境地，系统无法顺利启动。于是有了 initramfs 根文件系统，其中包含必要的设备驱动和工具，boot loader 加载 initramfs 到内存中，内核会将其挂载到根目录 /，然后运行 /init 脚本，挂载真正的磁盘根文件系统，最后通过 exec chroot . /sbin/init 命令来将根目录切换到挂载了实际磁盘文件系统中，并执行 /sbin/init 程序来启动系统中的其他进程和服务。

这里借助 BusyBox 构建极简 initramfs，提供基本的用户态可执行程序。

https://www.busybox.net/

下了最新版本 busybox-1.33.1.tar.bz2

#### 配置

编译成静态链接：

```
Settings  --->
  [*] Build static binary (no shared libs)
```

#### 编译

安装到 `_install` 目录：

```
make -j$(nproc) && make install
```

#### 制作 initramfs

```
cd ./_install
mkdir dev 
sudo mknod dev/console c 5 1
sudo mknod dev/ram b 1 0 
touch init
chmod +x init
```

init 初始化脚本的内容为：

```
#!/bin/sh
echo "INIT SCRIPT"
mkdir /proc
mkdir /sys
mount -t proc none /proc
mount -t sysfs none /sys
mkdir /tmp
mount -t tmpfs none /tmp
mknod -m 666 /dev/ttyS0 c 4 64
echo -e "\nThis boot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
setsid cttyhack /bin/sh
exec /bin/sh
```

打包成 cpio 格式：
```
find . -print0 | cpio --null -ov --format=newc | gzip -9 >  ../initramfs.cpio.gz
```

## 3. QEMU 启动一下试试

```
qemu-system-x86_64 \
  -s \
  -kernel ~/linux-5.13.12/arch/x86_64/boot/bzImage \
  -initrd  ~/busybox-1.33.1/initramfs.cpio.gz \
  --append "nokaslr root=/dev/ram init=/init console=ttyS0" \
  -nographic 
```

成功启动：

```
......
[    2.516738] Run /init as init process
INIT SCRIPT
[    2.543858] tsc: Refined TSC clocksource calibration: 2499.989 MHz
[    2.545084] clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x24092f7b636, max_idle_ns: 440795303187 ns
[    2.547960] clocksource: Switched to clocksource tsc

This boot took 2.55 seconds

/ # uname -a
Linux (none) 5.13.12 #3 SMP Thu Aug 26 15:20:22 CST 2021 x86_64 GNU/Linux
/ #
```

## 4. 如何 debug 呢？

首先在 `~/linux-5.13.12/.gdbinit` 中添加：
```
	add-auto-load-safe-path ~/linux-5.13.12/scripts/gdb/vmlinux-gdb.py
```

在`~/linux-5.13.12/`目录下，通过 gdb 连上去：
```
:~$ gdb vmlinux
(gdb) target remote:1234
Remote debugging using :1234
0xffffffff81bf73ee in native_safe_halt () at ./arch/x86/include/asm/irqflags.h:51
51		asm volatile("sti; hlt": : :"memory");
(gdb)

```

设置个断点：
```
((gdb) b cmdline_proc_show
Note: breakpoint 1 also set at pc 0xffffffff813d2140.
Breakpoint 2 at 0xffffffff813d2140: file fs/proc/cmdline.c, line 8.
(gdb) c
Continuing.
```

在 QEMU 虚拟机中执行 `cat /proc/cmdline` 就会触发：
```
(gdb) c
Continuing.

Breakpoint 1, cmdline_proc_show (m=0xffff888005253690, v=0x1 <fixed_percpu_data+1>) at fs/proc/cmdline.c:8
8	{
(gdb) bt
#0  cmdline_proc_show (m=0xffff888005253690, v=0x1 <fixed_percpu_data+1>) at fs/proc/cmdline.c:8
#1  0xffffffff81349a50 in seq_read_iter (iocb=0xffffc90000603d28, iter=0xffffc90000603d00) at fs/seq_file.c:230
#2  0xffffffff813c755e in proc_reg_read_iter (iocb=<optimized out>, iter=<optimized out>) at fs/proc/inode.c:300
#3  0xffffffff81358897 in call_read_iter (file=0xffff888004a55600, iter=0xffffc90000603d00, kio=0xffffc90000603d28)
    at ./include/linux/fs.h:2108
#4  generic_file_splice_read (in=0xffff888004a55600, ppos=0xffffc90000603de8, pipe=<optimized out>, len=<optimized out>,
    flags=<optimized out>) at fs/splice.c:311
#5  0xffffffff81358d61 in do_splice_to (in=in@entry=0xffff888004a55600, ppos=ppos@entry=0xffffc90000603de8,
    pipe=pipe@entry=0xffff888005207f00, len=65536, len@entry=16777216, flags=flags@entry=0) at fs/splice.c:796
#6  0xffffffff81358e4c in splice_direct_to_actor (in=in@entry=0xffff888004a55600, sd=sd@entry=0xffffc90000603e30,
    actor=actor@entry=0xffffffff813591d0 <direct_splice_actor>) at fs/splice.c:870
#7  0xffffffff81359049 in do_splice_direct (in=in@entry=0xffff888004a55600, ppos=ppos@entry=0xffffc90000603eb0,
    out=out@entry=0xffff888004a55200, opos=opos@entry=0xffffc90000603eb8, len=len@entry=16777216, flags=flags@entry=0)
    at fs/splice.c:979
#8  0xffffffff8131b183 in do_sendfile (out_fd=out_fd@entry=1, in_fd=in_fd@entry=3, ppos=ppos@entry=0x0 <fixed_percpu_data>,
    count=count@entry=16777216, max=2147483647, max@entry=0) at fs/read_write.c:1260
#9  0xffffffff8131b796 in __do_sys_sendfile64 (count=16777216, offset=0x0 <fixed_percpu_data>, in_fd=3, out_fd=1)
    at fs/read_write.c:1325
#10 __se_sys_sendfile64 (count=16777216, offset=0, in_fd=3, out_fd=1) at fs/read_write.c:1311
#11 __x64_sys_sendfile64 (regs=<optimized out>) at fs/read_write.c:1311
#12 0xffffffff81be4c80 in do_syscall_64 (nr=<optimized out>, regs=0xffffc90000603f58) at arch/x86/entry/common.c:47
#13 0xffffffff81c0007c in entry_SYSCALL_64 () at arch/x86/entry/entry_64.S:112
#14 0x0000000000000000 in ?? ()
```


## 5. 我想 debug epoll，还需要配置什么？

需要配置一下网络，这里我配置了 host-only 模式。

####  首先在宿主机上设置网桥和 TAP

设置网桥

```
sudo ip link add br0 type bridge
```

为网桥设置 ip

```
sudo ip addr add 192.168.100.50/24 brd 192.168.100.255 dev br0
```

创建 TAP 接口
```
sudo ip tuntap add mode tap user $(whoami)
ip tuntap show
````

将 TAP 接口添加到网桥

```
sudo ip link set tap0 master br0
```

启起来

```
sudo ip link set dev br0 up
sudo ip link set dev tap0 up
```

#### 然后配置 busybox 的网络

添加网卡驱动 e1000

```
cd ./_install
mkdir -p lib/modules
cp ~/linux-5.13.12/drivers/net/ethernet/intel/e1000/e1000.ko lib/modules/
```

`init` 脚本中加载驱动，并且配置网络
```

insmod /lib/modules/e1000.ko
ifconfig lo 127.0.0.1 netmask 255.0.0.0 up
ifconfig eth0 192.168.100.224 netmask 255.255.255.0 broadcast 192.168.100.255 up

```

重新打包  initramfs

#### 最后在 QEMU 启动时配置网络设备

```
-netdev tap,ifname=tap0,id=tap0,script=no,downscript=no -device e1000,netdev=tap0
```


## 6. debug 一下 epoll 的回调

一直对 epoll 的回调不太清楚，有了这个工具，debug 一下。

设置断点 `ep_poll_callback`：

```plain
#0  ep_poll_callback (wait=0xffff88800368a410, mode=1, sync=16, key=0xc3) at fs/eventpoll.c:1128
#1  0xffffffff810fe25e in __wake_up_common (wq_head=wq_head@entry=0xffff8880046dbac0, mode=mode@entry=1,
    nr_exclusive=nr_exclusive@entry=1, wake_flags=wake_flags@entry=16, key=key@entry=0xc3, bookmark=bookmark@entry=0xffffc90000003970)
    at kernel/sched/wait.c:108
#2  0xffffffff810fe39c in __wake_up_common_lock (wq_head=0xffff8880046dbac0, mode=mode@entry=1, nr_exclusive=nr_exclusive@entry=1,
    wake_flags=wake_flags@entry=16, key=key@entry=0xc3) at kernel/sched/wait.c:138
#3  0xffffffff810fe690 in __wake_up_sync_key (wq_head=<optimized out>, mode=mode@entry=1, key=key@entry=0xc3)
    at kernel/sched/wait.c:205
#4  0xffffffff81993a9b in sock_def_readable (sk=0xffff88800576e900) at net/core/sock.c:2925
#5  0xffffffff81a72c1f in tcp_data_ready (sk=sk@entry=0xffff88800576e900) at net/ipv4/tcp_input.c:4938
#6  0xffffffff81a73f64 in tcp_data_queue (sk=sk@entry=0xffff88800576e900, skb=skb@entry=0xffff888005778800)
    at net/ipv4/tcp_input.c:5008
#7  0xffffffff81a74760 in tcp_rcv_established (sk=sk@entry=0xffff88800576e900, skb=skb@entry=0xffff888005778800)
    at net/ipv4/tcp_input.c:5896
#8  0xffffffff81a82718 in tcp_v4_do_rcv (sk=sk@entry=0xffff88800576e900, skb=skb@entry=0xffff888005778800) at net/ipv4/tcp_ipv4.c:1694
#9  0xffffffff81a84c1d in tcp_v4_rcv (skb=0xffff888005778800) at net/ipv4/tcp_ipv4.c:2077
#10 0xffffffff81a524a0 in ip_protocol_deliver_rcu (net=0xffffffff828e44c0 <init_net>, skb=0xffff888005778800,
    protocol=<optimized out>) at net/ipv4/ip_input.c:204
#11 0xffffffff81a52668 in ip_local_deliver_finish (net=<optimized out>, sk=<optimized out>, skb=<optimized out>)
    at ./include/linux/skbuff.h:2539
#12 0xffffffff81a526f2 in NF_HOOK (sk=0x0 <fixed_percpu_data>, pf=2 '\002', hook=1, in=<optimized out>, out=0x0 <fixed_percpu_data>,
    okfn=0xffffffff81a52620 <ip_local_deliver_finish>, skb=0xffff888005778800, net=0xffffffff828e44c0 <init_net>)
    at ./include/linux/netfilter.h:301
#13 NF_HOOK (pf=2 '\002', sk=0x0 <fixed_percpu_data>, out=0x0 <fixed_percpu_data>, okfn=0xffffffff81a52620 <ip_local_deliver_finish>,
    in=<optimized out>, skb=0xffff888005778800, net=0xffffffff828e44c0 <init_net>, hook=1) at ./include/linux/netfilter.h:295
#14 ip_local_deliver (skb=0xffff888005778800) at net/ipv4/ip_input.c:252
#15 0xffffffff81a528b9 in dst_input (skb=<optimized out>) at ./include/linux/skbuff.h:975
#16 ip_sublist_rcv_finish (head=head@entry=0xffffc90000003c48) at net/ipv4/ip_input.c:551
#17 0xffffffff81a52a3f in ip_list_rcv_finish (sk=0x0 <fixed_percpu_data>, head=0xffffc90000003cd8, net=0xffffffff828e44c0 <init_net>)
    at net/ipv4/ip_input.c:601
#18 ip_sublist_rcv (head=head@entry=0xffffc90000003cd8, dev=dev@entry=0xffff8880035ee000, net=net@entry=0xffffffff828e44c0 <init_net>)
    at net/ipv4/ip_input.c:609
#19 0xffffffff81a52cb4 in ip_list_rcv (head=0xffffc90000003d50, pt=<optimized out>, orig_dev=<optimized out>)
    at net/ipv4/ip_input.c:644
#20 0xffffffff819b8e88 in __netif_receive_skb_list_ptype (orig_dev=0xffff8880035ee000, pt_prev=0xffffffff8296aa80 <ip_packet_type>,
    head=0xffffc90000003d50) at net/core/dev.c:5505
#21 __netif_receive_skb_list_core (head=head@entry=0xffff8880035eecf8, pfmemalloc=pfmemalloc@entry=false) at net/core/dev.c:5550
#22 0xffffffff819b9051 in __netif_receive_skb_list (head=0xffff8880035eecf8) at net/core/dev.c:5602
#23 netif_receive_skb_list_internal (head=head@entry=0xffff8880035eecf8) at net/core/dev.c:5712
#24 0xffffffff819b922e in gro_normal_list (napi=napi@entry=0xffff8880035eebf0) at net/core/dev.c:5866
#25 0xffffffff819ba091 in gro_normal_list (napi=0xffff8880035eebf0) at net/core/dev.c:6588
#26 napi_complete_done (n=0xffff8880035eebf0, work_done=<optimized out>) at net/core/dev.c:6588
#27 0xffffffffa000421a in ?? ()
#28 0xffffffff810c30d2 in __rcu_read_unlock () at ./include/linux/rcupdate.h:710
#29 rcu_read_unlock () at ./include/linux/rcupdate.h:710
#30 __queue_work (cpu=91230208, wq=0x40, work=0xffff8880035eebf0) at kernel/workqueue.c:1502
#31 0xffffffff819ba5d1 in __napi_poll (n=n@entry=0xffff8880035eebf0, repoll=repoll@entry=0xffffc90000003f1f) at net/core/dev.c:7008
#32 0xffffffff819baaaf in napi_poll (repoll=0xffffc90000003f30, n=0xffff8880035eebf0) at net/core/dev.c:7075
#33 net_rx_action (h=<optimized out>) at net/core/dev.c:7162
#34 0xffffffff81e000e0 in __do_softirq () at kernel/softirq.c:559
#35 0xffffffff810a91a4 in invoke_softirq () at kernel/softirq.c:433
#36 __irq_exit_rcu () at kernel/softirq.c:637
#37 irq_exit_rcu () at kernel/softirq.c:649
#38 0xffffffff81be611d in common_interrupt (regs=0xffffffff82603d68, error_code=<optimized out>) at arch/x86/kernel/irq.c:240
#39 0xffffffff81c00cde in asm_common_interrupt () at ./arch/x86/include/asm/idtentry.h:629
#40 0x0000000000000000 in ?? ()
```

调用栈一目了然。

---------------------------------
update 9月24日

1. 打开 Enable full Section mismatch analysis
```
Kernel hacking  --->
  Compile-time checks and compiler options  --->
    [*] Enable full Section mismatch analysis

```

打开编译选项CONFIG_DEBUG_SECTION_MISMATCH，如果关闭此选项，那么内核中仅仅被调用一次的函数会被编译优化为内联函数，不利于调试

2. 编辑Makefile，将其中的-O2 替还为 -O1
```
export KBUILD_USERCFLAGS := -Wall -Wmissing-prototypes -Wstrict-prototypes \
                 -g -O1 -fomit-frame-pointer -std=gnu89
KBUILD_HOSTCXXFLAGS := -Wall -g -O1 $(HOST_LFS_CFLAGS) $(HOSTCXXFLAGS)

ifdef CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE
KBUILD_CFLAGS += -O1
else ifdef CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE_O3
KBUILD_CFLAGS += -O3
else ifdef CONFIG_CC_OPTIMIZE_FOR_SIZE
KBUILD_CFLAGS += -Os
endif
```
