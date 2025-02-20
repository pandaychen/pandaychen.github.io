---
layout:     post
title:      gopsutil 使用与分析
subtitle:   一个跨平台的采集库 && windows API 实现
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

##  0x01 应用实战

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

##  0x02    参考
-   [Linux Performance](http://www.brendangregg.com/linuxperf.html)
-	[psutil for golang](https://github.com/shirou/gopsutil)
-	[Go 每日一库之 gopsutil](https://darjun.github.io/2020/04/05/godailylib/gopsutil/)
-	[A journey into the Linux proc filesystem](https://fernandovillalba.substack.com/p/a-journey-into-the-linux-proc-filesystem)
- 	[HIDS数据采集项总结](https://l0n9w4y.cc/posts/72019/)