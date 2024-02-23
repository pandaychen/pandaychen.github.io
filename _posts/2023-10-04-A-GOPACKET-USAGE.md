---
layout:     post
title:      DPI with go：构建一个流量分析系统
subtitle:   golang 的包处理：gopacket
date:       2023-10-04
author:     pandaychen
catalog:    true
tags:
    - DPI
    - gopacket
---


##  0x00    前言
项目中需要实现一些基本的流量分析功能，借用了 gopacket 库，该库是 libpcap 和 npcap 的 go 封装，提供了更方便的 go 语言操作接口。通常网络抓包有以下几个步骤：
1.  枚举主机上网络设备的接口
2.  针对某一网口进行抓包
3.  解析数据包的 mac 层、ip 层、tcp/udp 层字段等
4.  ip 分片重组，或 tcp 分段重组成上层协议如 http 协议的数据
5.  对上层协议进行头部解析和负载部分解析


####    应用场景
-   网络流量分析：对网络设备流量进行实时采集以及数据包分析
-   伪造数据包发送
-   离线 pcap 文件的读取和写入

1.	从网络接口收集数据包（tcpdump）
2.	重新组装 TCP 流（Wireshark）

####    libpcap 原理
![libpcap](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/libpcap/arch.png)

回顾下 libpcap 包处理路径，libpcap 包捕获机制是在数据链路层增加一个旁路处理，并不干扰系统自身的网路协议栈的处理，对发送和接收的数据包通过 Linux 内核做过滤和缓冲处理，最后直接传递给上层应用程序。 libpcap 在捕获到达网卡的数据包后绕开了传统 linux 协议栈处理，直接使用链路层 `PF_PACKET` 协议族原始套接字方式向用户空间传递报文

libpcap 主要由网络分接头（Network Tap）和数据过滤器（Packet Filter）构成，网络分接头从网络设备驱动程序（NIC driver）中收集数据拷贝，过滤器决定是否接收该数据包。Libpcap 的工作原理可以描述为，当一个数据包到达网卡时，通过网络分接口（即旁路机制）将数据包发给 BPF（BSD Packet Filter）过滤器，匹配通过的数据包可以被 libpcap 利用创建的套接字 PF_PACKET 从链路层驱动程序中获得。进而在用户空间提供独立于系统的用户级 API 接口，用户程序获取报文后可以做相关处理

##  0x01    gopacket 基础
以 TCP 报文为例，其在协议栈上的格式如下图所示，从 libpcap 拿到的原始数据是 `Ethernet` 层，然后逐层向上面解码，最终拿到开发者想要的数据（协议栈层次、应用数据等）
![stack](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/protocol/tcp-ip-stack.gif)

所以，需要熟练掌握如下知识点：
-   协议（报文）格式，比如 IP 层的 protocol 字段标识了其上层是何种协议，从而实现解析报文
-   TCP 会话重组：由于 TCP 是流式协议，完整的解析 TCP 需要实现重组功能

gopacket 的子包（主要）功能如下：

-   layers：包含内置于 gopacket 的用于解码数据包协议的逻辑，常用
-   pcap：使用 libpcap 从网络读取数据包的 C 绑定，可以用来读取 / 写入 pcap 数据包
-   pfring：使用 `PF_RING` 从网络读取数据包的 C 绑定
-   afpacket：从网络上读取数据包的 Linux `AF_PACKET` 的 C 绑定
-   tcpassembly：TCP 流重组，建议使用 [reassembly](https://github.com/google/gopacket/tree/master/reassembly)，这个库较新

####    基础示例
基础使用可以参考 [Basic_Usage](https://pkg.go.dev/github.com/google/gopacket@v1.1.19#hdr-Basic_Usage)、[examples](https://github.com/google/gopacket/tree/master/examples) 下面的代码

例子 1：设置过滤器 `tcp and port 80` 抓包

```go
var (
    device       string = "eth1"
    snapshot_len int32  = 1024
    promiscuous  bool   = false
    err          error
    timeout      time.Duration = 30 * time.Second
    handle       pcap.Handle
)

func main() {
    // Open device
    handle, err := pcap.OpenLive(device, snapshot_len, promiscuous, timeout)
    if err != nil {
        log.Fatal(err)
    }
    defer handle.Close()

    // Set filter
    var filter string = "tcp and port 80"
    err = handle.SetBPFFilter(filter)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Only capturing TCP port 80 packets.")

    packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
for packet := range packetSource.Packets() {
        // Do something with a packet here.
        fmt.Println(packet)
    }

}
```

例子 2：调用 `layers` 解码，作者已经提供了诸如 LayerTypeEthernet/IP/UDP/TCP 等众多 layer 创建了相应类型，参考 [layers](https://github.com/google/gopacket/tree/master/layers)；以 dns 协议为例，其格式定义 [为](https://github.com/google/gopacket/blob/master/layers/dns.go)

```GO
// BaseLayer is a convenience struct which implements the LayerData and
// LayerPayload functions of the Layer interface.
type BaseLayer struct {
	// Contents is the set of bytes that make up this layer.  IE: for an
	// Ethernet packet, this would be the set of bytes making up the
	// Ethernet frame.
	Contents []byte
	// Payload is the set of bytes contained by (but not part of) this
	// Layer.  Again, to take Ethernet as an example, this would be the
	// set of bytes encapsulated by the Ethernet protocol.
	Payload []byte
}

// DNS is specified in RFC 1034 / RFC 1035
// +---------------------+
// |        Header       |
// +---------------------+
// |       Question      | the question for the name server
// +---------------------+
// |        Answer       | RRs answering the question
// +---------------------+
// |      Authority      | RRs pointing toward an authority
// +---------------------+
// |      Additional     | RRs holding additional information
// +---------------------+
//
//  DNS Header
//  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
//  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
//  |                      ID                       |
//  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
//  |QR|   Opcode  |AA|TC|RD|RA|   Z    |   RCODE   |
//  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
//  |                    QDCOUNT                    |
//  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
//  |                    ANCOUNT                    |
//  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
//  |                    NSCOUNT                    |
//  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
//  |                    ARCOUNT                    |
//  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+

// DNS contains data from a single Domain Name Service packet.
type DNS struct {
	BaseLayer

	// Header fields
	ID     uint16
	QR     bool
	OpCode DNSOpCode

	AA bool  // Authoritative answer
	TC bool  // Truncated
	RD bool  // Recursion desired
	RA bool  // Recursion available
	Z  uint8 // Reserved for future use

	ResponseCode DNSResponseCode
	QDCount      uint16 // Number of questions to expect
	ANCount      uint16 // Number of answers to expect
	NSCount      uint16 // Number of authorities to expect
	ARCount      uint16 // Number of additional records to expect

	// Entries
	Questions   []DNSQuestion
	Answers     []DNSResourceRecord
	Authorities []DNSResourceRecord
	Additionals []DNSResourceRecord

	// buffer for doing name decoding.  We use a single reusable buffer to avoid
	// name decoding on a single object via multiple DecodeFromBytes calls
	// requiring constant allocation of small byte slices.
	buffer []byte
}
```

数据包解码的代码如下，其中的核心是 `packet.Layer`[方法](https://github.com/google/gopacket/blob/master/packet.go#L72)：

```go
var (
    device      string = "eth0"
    snapshotLen int32  = 1024
    promiscuous bool   = false
    err         error
    timeout     time.Duration = 30 * time.Second
    handle      pcap.Handle
)

func main() {
    // Open device
	// 对网卡流量进行实时捕获
    handle, err := pcap.OpenLive(device, snapshotLen, promiscuous, timeout)
    if err != nil {
        log.Fatal(err)
    }

	// 需要关闭
    defer handle.Close()

    packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
	// packetSource.Packets 返回一个 channel
    for packet := range packetSource.Packets() {
        printPacketInfo(packet)
    }
}

func printPacketInfo(packet gopacket.Packet) {
    // Let’s see if the packet is an ethernet packet
    ethernetLayer := packet.Layer(layers.LayerTypeEthernet)
    if ethernetLayer != nil {
        fmt.Println("Ethernet layer detected.")
        ethernetPacket, _ := ethernetLayer.(*layers.Ethernet)
        fmt.Println("Source MAC:", ethernetPacket.SrcMAC)
        fmt.Println("Destination MAC:", ethernetPacket.DstMAC)
        // Ethernet type is typically IPv4 but could be ARP or other
        fmt.Println("Ethernet type:", ethernetPacket.EthernetType)
        fmt.Println()
    }

    // Let’s see if the packet is IP (even though the ether type told us)
    ipLayer := packet.Layer(layers.LayerTypeIPv4)
    if ipLayer != nil {
        fmt.Println("IPv4 layer detected.")
        ip, _ := ipLayer.(*layers.IPv4)

        // IP layer variables:
        // Version (Either 4 or 6)
        // IHL (IP Header Length in 32-bit words)
        // TOS, Length, Id, Flags, FragOffset, TTL, Protocol (TCP?),
        // Checksum, SrcIP, DstIP
        fmt.Printf("From %s to %s\n", ip.SrcIP, ip.DstIP)
        fmt.Println("Protocol:", ip.Protocol)
        fmt.Println()
    }

    // Let’s see if the packet is TCP
    tcpLayer := packet.Layer(layers.LayerTypeTCP)
    if tcpLayer != nil {
        fmt.Println("TCP layer detected.")
		 // Get actual TCP data from this layer
        tcp, _ := tcpLayer.(*layers.TCP)

        // TCP layer variables:
        // SrcPort, DstPort, Seq, Ack, DataOffset, Window, Checksum, Urgent
        // Bool flags: FIN, SYN, RST, PSH, ACK, URG, ECE, CWR, NS
        fmt.Printf("From port %d to %d\n", tcp.SrcPort, tcp.DstPort)
        fmt.Println("Sequence number:", tcp.Seq)
        fmt.Println()
    }

    // Iterate over all layers, printing out each layer type
    fmt.Println("All packet layers:")
    for _, layer := range packet.Layers() {
        fmt.Println("-", layer.LayerType())
    }

    // When iterating through packet.Layers() above,
    // if it lists Payload layer then that is the same as
    // this applicationLayer. applicationLayer contains the payload
    applicationLayer := packet.ApplicationLayer()
    if applicationLayer != nil {
        fmt.Println("Application layer/Payload found.")
        fmt.Printf("%s\n", applicationLayer.Payload())

        // Search for a string inside the payload
        if strings.Contains(string(applicationLayer.Payload()), "HTTP") {
            fmt.Println("HTTP found!")
        }
    }

    // Check for errors
    if err := packet.ErrorLayer(); err != nil {
        fmt.Println("Error decoding some part of the packet:", err)
    }
}
```

##  0x02    代码分析
[`Packet`](https://github.com/google/gopacket/blob/master/packet.go#L56C6-L56C13) 是数据解码器的公共接口实现，`packet` 作为其 [实例化](https://github.com/google/gopacket/blob/master/packet.go#L106)，gopacket 提供了这几种内置解码器：

-	`eagerPacket`：
-	[`lazyPacket`](https://github.com/google/gopacket/blob/master/packet.go#L495)：

1、`Packet`

```go
// Packet is the primary object used by gopacket.  Packets are created by a
// Decoder's Decode call.  A packet is made up of a set of Data, which
// is broken into a number of Layers as it is decoded.
type Packet interface {
	//// Functions for outputting the packet as a human-readable string:
	//// ------------------------------------------------------------------
	// String returns a human-readable string representation of the packet.
	// It uses LayerString on each layer to output the layer.
	String() string
	// Dump returns a verbose human-readable string representation of the packet,
	// including a hex dump of all layers.  It uses LayerDump on each layer to
	// output the layer.
	Dump() string

	//// Functions for accessing arbitrary packet layers:
	//// ------------------------------------------------------------------
	// Layers returns all layers in this packet, computing them as necessary
	Layers() []Layer
	// Layer returns the first layer in this packet of the given type, or nil
	Layer(LayerType) Layer
	// LayerClass returns the first layer in this packet of the given class,
	// or nil.
	LayerClass(LayerClass) Layer

	//// Functions for accessing specific types of packet layers.  These functions
	//// return the first layer of each type found within the packet.
	//// ------------------------------------------------------------------
	// LinkLayer returns the first link layer in the packet
	LinkLayer() LinkLayer
	// NetworkLayer returns the first network layer in the packet
	NetworkLayer() NetworkLayer
	// TransportLayer returns the first transport layer in the packet
	TransportLayer() TransportLayer
	// ApplicationLayer returns the first application layer in the packet
	ApplicationLayer() ApplicationLayer
	// ErrorLayer is particularly useful, since it returns nil if the packet
	// was fully decoded successfully, and non-nil if an error was encountered
	// in decoding and the packet was only partially decoded.  Thus, its output
	// can be used to determine if the entire packet was able to be decoded.
	ErrorLayer() ErrorLayer

	//// Functions for accessing data specific to the packet:
	//// ------------------------------------------------------------------
	// Data returns the set of bytes that make up this entire packet.
	Data() []byte
	// Metadata returns packet metadata associated with this packet.
	Metadata() *PacketMetadata
}
```

2、`Layer`：抽象了协议层次，gopacket 提供了如下几类：

-	`LinkLayer`： the packet layer corresponding to TCP/IP layer 1 (OSI layer 2)
-	`NetworkLayer`：the packet layer corresponding to TCP/IP layer 2 (OSI layer 3)，实例化参考 [ipv4](https://github.com/google/gopacket/blob/master/layers/ip4.go#L63C27-L63C28)
-	`TransportLayer`：is the packet layer corresponding to the TCP/IP layer 3 (OSI layer 4)，实例化参考 [tcp](https://github.com/google/gopacket/blob/master/layers/tcp.go#L331)
-	`ApplicationLayer`：the packet layer corresponding to the TCP/IP layer 4 (OSI layer 7), also known as the packet payload
-	`ErrorLayer`：a packet layer created when decoding of the packet has failed

```GO
// Layer represents a single decoded packet layer (using either the
// OSI or TCP/IP definition of a layer).  When decoding, a packet's data is
// broken up into a number of layers.  The caller may call LayerType() to
// figure out which type of layer they've received from the packet.  Optionally,
// they may then use a type assertion to get the actual layer type for deep
// inspection of the data.
type Layer interface {
	// LayerType is the gopacket type for this layer.
	LayerType() LayerType
	// LayerContents returns the set of bytes that make up this layer.
	LayerContents() []byte
	// LayerPayload returns the set of bytes contained within this layer, not
	// including the layer itself.
	LayerPayload() []byte
}
```

3、[`LayerType`](https://github.com/google/gopacket/blob/master/layertype.go)


##	0x03	TCP 流重组
TCP 流重组的代码在 [此](https://github.com/google/gopacket/blob/master/reassembly/tcpassembly.go)

####	httpassembly
这里简单分析下 [httpassembly](https://github.com/google/gopacket/blob/master/examples/httpassembly/main.go) 的实现

1、核心结构

```go
// Build a simple HTTP request parser using tcpassembly.StreamFactory and tcpassembly.Stream interfaces

// httpStreamFactory implements tcpassembly.StreamFactory
type httpStreamFactory struct{}

// httpStream will handle the actual decoding of http requests.
type httpStream struct {
	net, transport gopacket.Flow
	r              tcpreader.ReaderStream
}

func (h *httpStreamFactory) New(net, transport gopacket.Flow) tcpassembly.Stream {
	hstream := &httpStream{
		net:       net,
		transport: transport,
		r:         tcpreader.NewReaderStream(),
	}
	go hstream.run() // Important... we must guarantee that data from the reader stream is read.

	// ReaderStream implements tcpassembly.Stream, so we can return a pointer to it.
	return &hstream.r
}
```

而重组的逻辑如下：

1.	收包并组装 `AssembleWithTimestamp`
2.	超时清理 `FlushOlderThan`

```GO
func main() {
	// ...
	streamFactory := &httpStreamFactory{}
	streamPool := tcpassembly.NewStreamPool(streamFactory)
	assembler := tcpassembly.NewAssembler(streamPool)

	log.Println("reading in packets")
	// Read in packets, pass to assembler.
	packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
	packets := packetSource.Packets()
	ticker := time.Tick(time.Minute)
	for {
		select {
		case packet := <-packets:
			// A nil packet indicates the end of a pcap file.
			if packet == nil {
				return
			}
			if *logAllPackets {
				log.Println(packet)
			}
			if packet.NetworkLayer() == nil || packet.TransportLayer() == nil || packet.TransportLayer().LayerType() != layers.LayerTypeTCP {
				log.Println("Unusable packet")
				continue
			}
			tcp := packet.TransportLayer().(*layers.TCP)
			assembler.AssembleWithTimestamp(packet.NetworkLayer().NetworkFlow(), tcp, packet.Metadata().Timestamp)

		case <-ticker:
			// Every minute, flush connections that haven't seen activity in the past 2 minutes.
			assembler.FlushOlderThan(time.Now().Add(time.Minute * -2))
		}
	}
}
```

####	流重组的实现分析
快速回顾一下，TCP 流是网络上两台主机之间交换的连续数据流。为了允许网络适应不同的带宽，网络堆栈将每个 TCP 流拆分为多个数据包。由于底层 IP 网络不保证按顺序传送，因此数据包捕获可能包含每个流的重复或无序数据包，这也是 TCP 重组实现的重点问题

使用 gopacket 重组包需要实现两个接口 `interface`：
-	`Stream `：每个流代表一个重组的 TCP 流，并且是重组包将数据从 TCP 数据包传递给开发者的机制
-	`StreamFactory`：用于为每个 TCP 流构造新 Stream 的包装器（wrapper）


![assembly]()


##  0x04	参考
-   [Provides packet processing capabilities for Go](https://github.com/google/gopacket)
-   [[译] 利用 gopackage 进行包的捕获、注入和分析](https://colobu.com/2019/06/01/packet-capture-injection-and-analysis-gopacket/)
-   [tcp 重组](https://github.com/google/gopacket/blob/master/reassembly/tcpassembly.go)
-   [tcp 重组 - 实现（推荐）](https://github.com/google/gopacket/blob/master/reassembly/)
-   [网络流量抓包库 gopacket 介绍](https://blog.csdn.net/RA681t58CJxsgCkJ31/article/details/115152820)
-   [gopacket-GODOC](https://pkg.go.dev/github.com/google/gopacket#section-readme)
-   [Rebuilding Network Flows in Go](https://medium.com/a-bit-off/rebuilding-network-flows-in-gol-d89cfa0884ae)
-   [TCP/IP 详解 卷 1：协议（原书第 2 版）](https://book.douban.com/subject/26825411/)
-	[网络入侵检测系统之 Suricata(十一)--TCP 重组实现详解](https://zhuanlan.zhihu.com/p/393121010)
-	[stream-tcp-reassemble.c](https://doxygen.openinfosecfoundation.org/stream-tcp-reassemble_8c_source.html)
-	[爱奇艺网络流量分析引擎 QNSM 及其应用](https://www.infoq.cn/article/eelzsuyzo7rbjaxk6tau)
-	[Programmatically Analyze Packet Captures with GoPacket](https://www.akitasoftware.com/blog-posts/programmatically-analyze-packet-captures-with-gopacket)
-	[解析 http：parser](https://github.com/akitasoftware/akita-libs/blob/main/akinet/http/parser.go)
-	[tcp 重组实现：stream](https://github.com/akitasoftware/akita-cli/blob/main/pcap/stream.go)