# <center>DPDK学习笔记</center>

主要是根据官方推荐的文档顺序进行学习，学习路径为

+ Getting started Guide, 学习如何安装和编译dpdk，能够快速上手一些dpdk的应用

+ Programmer’s Guide。学习一些dpdk的思想和架构

+ API reference， 了解dpdk的一些API

+ Sample Applications User Guide，学习一些基于dpdk的application

## Getting Started Guide

### 系统配置和需求
一般dpdk不需要特别的BIOS设置，但是如果需要一些特殊的用途，可能需要BIOS，相关信息可查阅[相应文档](http://doc.dpdk.org/guides/linux_gsg/enable_func.html#enabling-additional-functionality)

系统启动之后建议设置大页，防止启动一段时间后内存碎片化难以分配有效的连续空间，对于2MB的大页和1GB大页，设置命令如下：
```shell
hugepages=1024
default_hugepagesz=1G hugepagesz=1G hugepages=4
```
对于64位的程序，建议使用1GB的大页

对于2MB的大页，在系统已经启动之后可以用以下命令分配大页：
```shell
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```
对于NUMA的机器，命令为:
```shell
echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 1024 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```
对于1GB的大页，系统启动之后没法重新分配大页。

让dpdk能使用大页，需要以下命令：
```shell
mkdir /mnt/huge
mount -t hugetlbfs nodev /mnt/huge
```
如果想要每次重启之后都能自动执行，可以在`/etc/fstab`文件中添加如下指令：
```shell
nodev /mnt/huge hugetlbfs defaults 0 0
nodev /mnt/huge_1GB hugetlbfs pagesize=1GB 0 0 (针对1GB的大页)
```
### Linux驱动
不同的PMDs（物理介质关联层接口）需要不同的内核驱动。对于网络端口，根据PMD的不同，需要绑定相应的内核驱动。

#### UIO
UIO是将设备的读写放到用户态空间，并且注册了一系列中断。标准的`uio_pci_generic`模式可以用以下指令加载：
```shell
sudo modprobe uio_pci_generic
```
对于一些不支持legacy interrupts（Legacy interrupts – Legacy or fixed interrupts refer to interrupts that use older bus technologies. With these technologies, interrupts are signaled by using one or more external pins that are wired “out-of-band,” that is, separately from the main lines of the bus）的设备，需要用`igb_uio`驱动，配置命令如下：
```shell
sudo modeprobe uio
sudo insmod kmod/igb_uio.ko
```
#### VFIO
对比UIO，VIFO是一种更加robust和secure的驱动，它依赖于IOMMU(输入输出内存管理单元)保护。使用VIFO的命令如下：
```shell
sudo modprobe vfio-pci
```
使用VFIO模式，bios和内核都要支持IO虚拟化才行。

#### 绑定/解绑内核模式下的网络端口
对于需要使用DPDK驱动的应用，必须保证PMD的驱动是`uio_pic_generic`,`igb_uio`或`vfio-pci`，然后利用dpdk绑定网卡。
绑定网卡可以用`/usertools/dpdk-devbind.py`脚本，如：
```shell
./usertools/dpdk-devbind.py --status

Network devices using DPDK-compatible driver
============================================
0000:82:00.0 '82599EB 10-GbE NIC' drv=uio_pci_generic unused=ixgbe
0000:82:00.1 '82599EB 10-GbE NIC' drv=uio_pci_generic unused=ixgbe

Network devices using kernel driver
===================================
0000:04:00.0 'I350 1-GbE NIC' if=em0  drv=igb unused=uio_pci_generic *Active*
0000:04:00.1 'I350 1-GbE NIC' if=eth1 drv=igb unused=uio_pci_generic
0000:04:00.2 'I350 1-GbE NIC' if=eth2 drv=igb unused=uio_pci_generic
0000:04:00.3 'I350 1-GbE NIC' if=eth3 drv=igb unused=uio_pci_generic

Other network devices
=====================
<none>
```
绑定deth1设备, ``04:00.1``, 利用uio_pci_generic驱动:
```shell
./usertools/dpdk-devbind.py --bind=uio_pci_generic 04:00.1
```
解绑``82:00.0``：
```shell
./usertools/dpdk-devbind.py --bind=ixgbe 82:00.0
```

### 编译、运行简单的应用
#### 编译一个简单的应用
编译dpdk程序之前，首先要设置以下的参数：
+ `RTE_SDK`——指向dpdk安装的目录
+ `RTE_TARGET`——指向dpdk目标环境
例如：
```shell
cd examples/helloworld/
export RTE_SDK=$HOME/DPDK
export RTE_TARGET=x86_64-native-linux-gcc

make
```
#### 运行一个简单的应用
运行应用前需要确保：
+ 大页分配完成
+ 驱动已经被加载

运行程序的时候，需要指定EAL(dpdk环境抽象层)的参数，如下：
```
./rte-app [-c COREMASK | -l CORELIST] [-n NUM] [-b <domain:bus:devid.func>] \
          [--socket-mem=MB,...] [-d LIB.so|DIR] [-m MB] [-r NUM] [-v] [--file-prefix] \
          [--proc-type <primary|secondary|auto>]
```
各个参数的意义如下：
+ `-c COREMASK` or `-l CORELIST`: An hexadecimal bit mask of the cores to run on. Note that core numbering can change between platforms and should be determined beforehand. The corelist is a set of core numbers instead of a bitmap core mask.
+ `-n NUM`: Number of memory channels per processor socket.
+ `-b \<domain:bus:devid.func\>`: Blacklisting of ports; prevent EAL from using specified PCI device (multiple -b options are allowed).
+ `--use-device`: use the specified Ethernet device(s) only. Use comma-separate [domain:]bus:devid.func values. Cannot be used with -b option.
+ `--socket-mem`: Memory to allocate from hugepages on specific sockets. In dynamic memory mode, this memory will also be pinned (i.e. not released back to the system until application closes).
+ `--socket-limit`: Limit maximum memory available for allocation on each socket. Does not support legacy memory mode.
+ `-d`: Add a driver or driver directory to be loaded. The application should use this option to load the pmd drivers that are built as shared libraries.
+ `-m MB`: Memory to allocate from hugepages, regardless of processor socket. It is recommended that --socket-mem be used instead of this option.
+ `-r NUM`: Number of memory ranks.
+ `-v`: Display version information on startup.
+ `--huge-dir`: The directory where hugetlbfs is mounted.
+ `mbuf-pool-ops-name`: Pool ops name for mbuf to use.
+ `--file-prefix`: The prefix text used for hugepage filenames.
+ `--proc-type`: The type of process instance.
+ `--vmware-tsc-map`: Use VMware TSC map instead of native RDTSC.
+ `--base-virtaddr`: Specify base virtual address.
+ `--vfio-intr`: Specify interrupt type to be used by VFIO (has no effect if VFIO is not used).
+ `--legacy-mem`: Run DPDK in legacy memory mode (disable memory reserve/unreserve at runtime, but provide more IOVA-contiguous memory).
+ `--single-file-segments`: Store memory segments in fewer files (dynamic memory mode only - does not affect legacy memory mode).

绑定CPU的时候，可以用命令`cat /proc/cpuinfo`来查看相关信息。

使用socket-mem选项可以为特定的插槽请求特定大小的内存。通过提供 `--socket-mem` 标志和每个插槽需要的内存数量来实现的，如 `--socket-mem=0,512` 用于在插槽1上预留512MB内存。 类似的，在4插槽系统上，如果只能在插槽0和2上分配1GB内存，则可以使用参数``–socket-mem=1024,0,1024`` 来实现。 如果DPDK无法在每个插槽上分配足够的内存，则EAL初始化失败。

为了提高性能，可以用`isolcpus`命令来使用Linux Core隔离。如dpdk程序运行在2、4、6逻辑核上，使用以下指令：
```shell
isolcpus=2,4,6
```
