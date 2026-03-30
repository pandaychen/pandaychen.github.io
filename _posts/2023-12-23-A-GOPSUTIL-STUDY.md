---
layout:     post
title:      gopsutil 使用与分析
subtitle:   一个跨平台的采集库实现与应用
date:       2024-08-04
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Golang
    - gopsutil
---

##	0x00	前言
gopsutil 屏蔽了各个系统之间的差异，具有非常好的可移植性

本文基于 [v4.24.7](https://github.com/shirou/gopsutil/releases/tag/v4.24.7) 分析


##	0x01	Linux下的proc应用

####	网络状态文件

1、`/proc/net/tcp`

```BASH
sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode
0: 0100007F:EA74 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 2819093322 1 ffff888313887200 100 0 0 10 0      
# ......
6: 0100007F:22B8 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 623457565 1 ffff88004f918740 750 0 0 2 -1
2: 00000000:8CA0 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 28740 1 ffff8881047d0980 100 0 0 10 0                     
19: XXXXXXXX:8CA0 2425570A:F464 01 00000080:00000000 01:00000008 00000000     0        0 3082231888 4 ffff88839b88d580 24 4 31 144 61
```

对应与`netstat -anpt`结果：

```BASH
[root@VM-X-X-X cmd]# netstat -napt|grep 36000
tcp        0      0 0.0.0.0:36000           0.0.0.0:*               LISTEN      4175/sshd: /usr/sbi 
tcp        0     64 X.X.X.X:36000     X.X.X.X:62564       ESTABLISHED 3811453/sshd: panda 
tcp6       0      0 :::36000                :::*                    LISTEN      4175/sshd: /usr/sbi 
```

`/proc/net/tcp`输出内容的核心字段：
-	`local_address`/`rem_address`：五元组信息，`00000000:8CA0`对应`0.0.0.0:36000`
-	`st`：网络连接状态，`0A`代表进程正监听端口，`01`代表已经建立的TCP连接
-	`inode`：inode号，如`28740`表示某进程监听socket（状态`0A`）的inode号，代表Linux系统中的一个文件系统对象包括文件、目录、设备文件、socket、管道等的元信息

```BASH
"01": "ESTABLISHED",
"02": "SYN_SENT",	
"03": "SYN_RECV",
"04": "FIN_WAIT1",
"05": "FIN_WAIT2",
"06": "TIME_WAIT",
"07": "CLOSE",
"08": "CLOSE_WAIT",
"09": "LAST_ACK",
"0A": "LISTEN",
"0B": "CLOSING"
```

2、`/proc/net/udp`

```BASH
sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode ref pointer drops            
4495: AF778609:0044 01708609:0043 01 00000000:00000000 00:00000000 00000000     0        0 10793 2 ffff88810d8b3440 0         
4750: 0100007F:0143 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 9959 2 ffff88810dbf0000 0     
```

####	/proc/${pid}/fd
记录了进程打开了文件句柄fd的快照信息，其中`0`、`1`、`2`表示标准输入、输出、错误，网络连接fd是以`socket:`开头的文件描述符，其中`[]`号内的是socket对应inode号，这样可以和网络状态文件`/proc/net/tcp`下的inode号进行关联

```BASH
[root@VM-X-X-X beat]# ll -arth /proc/4175/fd
total 0
dr-xr-xr-x 9 root root  0 Nov 28 13:25 ..
dr-x------ 2 root root  5 Nov 28 13:25 .
lrwx------ 1 root root 64 Nov 28 13:25 4 -> 'socket:[28742]'
lrwx------ 1 root root 64 Nov 28 13:25 3 -> 'socket:[28740]'
lrwx------ 1 root root 64 Nov 28 13:25 2 -> 'socket:[28731]'
lrwx------ 1 root root 64 Nov 28 13:25 1 -> 'socket:[28731]'
lr-x------ 1 root root 64 Nov 28 13:25 0 -> /dev/null
```

从上文看，可以通过`/proc/net/tcp`建立起五元组与inode的映射关系, 再从`/proc/pid/fd`建立起连接inode与进程的映射关系，这样通过inode号关联起系统内的进程与网络连接的信息，进而统计基于进程维度的流量采样数据，然后再基于`/proc/${pid}/cmdline`获取进程的启动参数等信息

```BASH
[root@VM-X-X-X tmp]# cat /proc/4175/cmdline 
sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
```

##  0x0 应用实战

先引入一个问题，已知 srcip、srcport，如何查找到该信息对应（占用）的进程名字（pid）？思路大致如下：
1.	通过 srcip、srcport 获取进程 pid
2.	通过 pid 获取进程名（进程路径）

####	Linux 实现

####  Windows 实现
1、通过查 TCP 连接表暴力破解的方法来实现第一个目标，参考 [getTCPConnections](https://github.com/shirou/gopsutil/blob/master/net/net_windows.go#L402)，获取到的结果 `ConnectionStat` 中包含 `Pid`

```GO
type ConnectionStat struct {
	Fd     uint32  `json:"fd"`
	Family uint32  `json:"family"`
	Type   uint32  `json:"type"`
	Laddr  Addr    `json:"localaddr"` 
	Raddr  Addr    `json:"remoteaddr"`
	Status string  `json:"status"`  //当前TCP连接的状态
	Uids   []int32 `json:"uids"`
	Pid    int32   `json:"pid"`   //pid
}


func getTCPConnections(family uint32) ([]ConnectionStat, error) {
	var (
		p    uintptr
		buf  []byte
		size uint32

		pmibTCPTable  pmibTCPTableOwnerPidAll
		pmibTCP6Table pmibTCP6TableOwnerPidAll
	)

	if family == 0 {
		return nil, fmt.Errorf("faimly must be required")
	}

	for {
		switch family {
		case kindTCP4.family:
			if len(buf) > 0 {
				pmibTCPTable = (*mibTCPTableOwnerPid)(unsafe.Pointer(&buf[0]))
				p = uintptr(unsafe.Pointer(pmibTCPTable))
			} else {
				p = uintptr(unsafe.Pointer(pmibTCPTable))
			}
		case kindTCP6.family:
			if len(buf) > 0 {
				pmibTCP6Table = (*mibTCP6TableOwnerPid)(unsafe.Pointer(&buf[0]))
				p = uintptr(unsafe.Pointer(pmibTCP6Table))
			} else {
				p = uintptr(unsafe.Pointer(pmibTCP6Table))
			}
		}

		err := getExtendedTcpTable(p,
			&size,
			true,
			family,
			tcpTableOwnerPidAll,
			0)
		if err == nil {
			break
		}
		if err != windows.ERROR_INSUFFICIENT_BUFFER {
			return nil, err
		}
		buf = make([]byte, size)
	}

	var (
		index, step int
		length      int
	)

	stats := make([]ConnectionStat, 0)
	switch family {
	case kindTCP4.family:
		index, step, length = getTableInfo(kindTCP4.filename, pmibTCPTable)
	case kindTCP6.family:
		index, step, length = getTableInfo(kindTCP6.filename, pmibTCP6Table)
	}

	if length == 0 {
		return nil, nil
	}

	for i := 0; i < length; i++ {
		switch family {
		case kindTCP4.family:
			mibs := (*mibTCPRowOwnerPid)(unsafe.Pointer(&buf[index]))
			ns := mibs.convertToConnectionStat()
			stats = append(stats, ns)
		case kindTCP6.family:
			mibs := (*mibTCP6RowOwnerPid)(unsafe.Pointer(&buf[index]))
			ns := mibs.convertToConnectionStat()
			stats = append(stats, ns)
		}

		index += step
	}
	return stats, nil
}
```

通过`getExtendedTcpTable`获取当前TCP连接表的信息，其中本质上调用了windows的`iphlpapi.dll`提供的`GetExtendedTcpTable`方法：

```GO
var (
	modiphlpapi             = windows.NewLazySystemDLL("iphlpapi.dll")
	procGetExtendedTCPTable = modiphlpapi.NewProc("GetExtendedTcpTable")
	procGetExtendedUDPTable = modiphlpapi.NewProc("GetExtendedUdpTable")
	procGetIfEntry2         = modiphlpapi.NewProc("GetIfEntry2")
)

func getExtendedTcpTable(pTcpTable uintptr, pdwSize *uint32, bOrder bool, ulAf uint32, tableClass tcpTableClass, reserved uint32) (errcode error) {
	r1, _, _ := syscall.Syscall6(procGetExtendedTCPTable.Addr(), 6, pTcpTable, uintptr(unsafe.Pointer(pdwSize)), getUintptrFromBool(bOrder), uintptr(ulAf), uintptr(tableClass), uintptr(reserved))
	if r1 != 0 {
		errcode = syscall.Errno(r1)
	}
	return
}
```

2、获取当前的TCP连接表后，可以获得这些基础信息，五元组、进程pid、当前TCP连接的状态，根据srcip、srcport扫表可以获取到对应的进程pid，具体代码可以参考[find_process.go](https://github.com/pandaychen/golang_in_action/blob/master/tun/find_process.go#L25)，由于是扫表查询，所以感觉这里性能会差一点

```go
func (m *mibTCP6RowOwnerPid) convertToConnectionStat() ConnectionStat {
	ns := ConnectionStat{
		Family: kindTCP6.family,
		Type:   kindTCP6.sockType,
		Laddr: Addr{
			IP:   parseIPv6HexString(m.UcLocalAddr),
			Port: uint32(decodePort(m.DwLocalPort)),
		},
		Raddr: Addr{
			IP:   parseIPv6HexString(m.UcRemoteAddr),
			Port: uint32(decodePort(m.DwRemotePort)),
		},
		Pid:    int32(m.DwOwningPid),
		Status: tcpStatuses[mibTCPState(m.DwState)],
	}

	return ns
}
```

其中TCP状态的定义如下：
```go
// tcpStatuses https://msdn.microsoft.com/en-us/library/windows/desktop/bb485761(v=vs.85).aspx
var tcpStatuses = map[mibTCPState]string{
	1:  "CLOSED",
	2:  "LISTEN",
	3:  "SYN_SENT",
	4:  "SYN_RECEIVED",
	5:  "ESTABLISHED",
	6:  "FIN_WAIT_1",
	7:  "FIN_WAIT_2",
	8:  "CLOSE_WAIT",
	9:  "CLOSING",
	10: "LAST_ACK",
	11: "TIME_WAIT",
	12: "DELETE",
}
```

3、最后，根据pid获取对应的归属进程信息（进程绝对路径，包含进程名），参考[ExeWithContext](https://github.com/shirou/gopsutil/blob/master/process/process_windows.go#L342)方法

```GO
func (p *Process) ExeWithContext(ctx context.Context) (string, error) {
	c, err := windows.OpenProcess(processQueryInformation, false, uint32(p.Pid))
	if err != nil {
		return "", err
	}
	defer windows.CloseHandle(c)
	buf := make([]uint16, syscall.MAX_LONG_PATH)
	size := uint32(syscall.MAX_LONG_PATH)
	if err := procQueryFullProcessImageNameW.Find(); err == nil { // Vista+
		ret, _, err := procQueryFullProcessImageNameW.Call(
			uintptr(c),
			uintptr(0),
			uintptr(unsafe.Pointer(&buf[0])),
			uintptr(unsafe.Pointer(&size)))
		if ret == 0 {
			return "", err
		}
		return windows.UTF16ToString(buf[:]), nil
	}
	// XP fallback
	ret, _, err := procGetProcessImageFileNameW.Call(uintptr(c), uintptr(unsafe.Pointer(&buf[0])), uintptr(size))
	if ret == 0 {
		return "", err
	}
	return common.ConvertDOSPath(windows.UTF16ToString(buf[:])), nil
}
```

其中，`procQueryFullProcessImageNameW`来自于`kernel32.dll`提供的方法：

```GO
Modkernel32 = windows.NewLazySystemDLL("kernel32.dll")
procQueryFullProcessImageNameW = Modkernel32.NewProc("QueryFullProcessImageNameW")
```

这里多提一下，上面大量使用了`unsafe.Pointer`，使用指针进行计算：

```go
type Foo struct {
        a int
        b string
}

func main() {
        var f Foo
        q := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&f))))
        *q = 10
        p := (*string)(unsafe.Pointer(uintptr(unsafe.Pointer(&f)) + unsafe.Offsetof(f.b)))
        *p = "hello" // 设置 f.b 的值，即使它是私有的
        fmt.Println(f.b, f.a)	// 输出hello 10
}
```

##	0x0	跨平台的兼容

####	CPU

`CPUPercent`在跨平台下的实现，TODO

####	内存

`MemoryPercent`在跨平台下的实现，TODO

##  0x0    参考
-   [Linux Performance](http://www.brendangregg.com/linuxperf.html)
-	[psutil for golang](https://github.com/shirou/gopsutil)
-	[Go 每日一库之 gopsutil](https://darjun.github.io/2020/04/05/godailylib/gopsutil/)
-	[A journey into the Linux proc filesystem](https://fernandovillalba.substack.com/p/a-journey-into-the-linux-proc-filesystem)
- 	[HIDS数据采集项总结](https://l0n9w4y.cc/posts/72019/)