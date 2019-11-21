# Linux内核 #

>linux内核时linux系统的一部分，其核心功能就是：<font color=red>管理硬件设备，供应用程序使用</font>。而现代计算机（无论是PC还是嵌入式系统）的标准组成，就是CPU、Memory（内存和外存）、输入输出设备、网络设备和其它的外围设备。所以为了管理这些设备，Linux内核提出了如下的架构。

![Linux内核架构](/assets/Linux内核架构.png)

根据内核的核心功能，Linux内核划分为5个子系统：
* ##### 进程调度（ Process Scheduler ）
* ##### 内存管理（ Memory Manager ）
* ##### 虚拟文件系统（ Virtual File System/VFS ）
* ##### 网络子系统（ Network ）
* ##### 进程间通信（ inter-Process Communication/IPC ）

## 进程调度 ##
