#### LWIP主进程工作

```c
/* LWIP 协议模拟了 TCP/IP 协议的分层思想，
表面上看 LWIP 也是有分层思想的，但从实现上看， LWIP 只在一个进程内实现了各个层次
的所有工作。具体如下： LWIP 完成相关初始化后，会阻塞在一个邮箱上，等待接收数据进
行处理。这个邮箱内的数据可能来自底层硬件驱动接收到的数据包，也可能来自应用程序。
当在该邮箱内取得数据后， LWIP 会对数据进行解析，然后再依次调用协议栈内部上层相关
处理函数处理数据。处理结束后， LWIP 继续阻塞在邮箱上等待下一批数据。
*/
```





**当数据在各层之间传递时，LWIP极力禁止数据的拷贝工作，因为会耗费大量的时间和内存**

```c
struct pbuf{
  	struct pbuf *next;
    void *payload;	// 指向该pbuf管理的数据的起始地址：紧跟在pbuf结构后的RAM,或者ROM上的某个地址
    u16_t tot_len;
    u16_t len;
    u8_t type;	// PBUF_RAM\PBUF_ROM\PBUF_PER\PBUF_POOL
    u8_t flags;
    u16_t ref;
};
```

| 应用层 |
| :----- |
| 传输层 |
| 网络层 |
| 链路层 |

LWIP没有在各层之间进行严格的划分，各层协议之间有交叉存取。

#### 链路层

netif描述一个硬件网络接口

```c
struct netif{
    struct netif *next;
    
    struct ip_addr ip_addr;	// IP地址
    struct ip_addr netmask;	// 子网掩码
    struct ip_addr gw;		// 网关地址
    
    err_t (*input)(struct pbuf *p, struct netif *inp);	// 调用这个函数从网卡取得一个数据包
    err_t (*output)(struct netif *netif, struct pbuf* p, struct ip_addr *ipaddr);	// IP层调用函数，向网卡发送一个数据包
    err_t (*linkoutput)(struct netif *netif, struct pbuf *p);	// ARP模块调用，向网卡发送一个数据包
   
    void *state;	// 用于指向用户关心的网卡信息
    u8_t hwaddr_len;	// 硬件地址长度，对于以太网就是MAC地址长度，为6个字节
    u8_t hwaddr[NETIF_MAX_HWADDR_LEN];	//MAC地址
    u16_t mtu;	//	一次可以传输的最大字节数，以太网一般设1500
    u8_t flags;	//	网卡状态信息标志位
    
    char name[2];	// 网络接口使用的设备驱动类型的种类
    u8_t num;		// 用来标识使用同种驱动类型的不同网络接口
};
```

![image-20200811100603518](D:\Saber_Workshop\Personal\Doc\Markdown_Note\Book\协议栈设计_LwIP笔记.assets\image-20200811100603518.png)

![image-20200811101404822](D:\Saber_Workshop\Personal\Doc\Markdown_Note\Book\协议栈设计_LwIP笔记.assets\image-20200811101404822.png)

![image-20200811101531933](D:\Saber_Workshop\Personal\Doc\Markdown_Note\Book\协议栈设计_LwIP笔记.assets\image-20200811101531933.png)

![image-20200811101633055](D:\Saber_Workshop\Personal\Doc\Markdown_Note\Book\协议栈设计_LwIP笔记.assets\image-20200811101633055.png)





#### LWIP数据包收发函数框架

low_level_input	low_level_output

LWIP应用系统包括三个进程：

* 上层应用程序进程
* LWIP协议栈进程
* 底层硬件数据包接收进程

| 目标MAC地址 | 源MAC地址 | 类型/长度                   | 数据        | 校验  |
| ----------- | --------- | --------------------------- | ----------- | ----- |
| 6字节       | 6字节     | 2字节(IP:0x0800/ARP:0x0806) | 46-1500字节 | 4字节 |

ps: 最大帧长1518字节，最小64字节

eth_hdr描述以太网数据包包头14个字节

```c
PACK_STRUCT_BEGIN	// 与编译器字对其相关的宏定义
	struct eth_hdr{
        PACK_STRUCT_FIELD(struct eth_addr dest);	// 目标MAC地址
        PACK_STRUCT_FIELD(struct eth_addr src);		// 源MAC地址
        PACK_STRUCT_FIELD(u16_t type);			   // 类型
    }PACK_STRUCT_STRUCT;
PACK_STRUCT_END
```

大端模式：某个半字或字数据的高位字节存在内存的低地址端，低位字节存放在内存的高地址端
小端模式：某个半字或字数据的 高位字节存在内存的高地址端，低位字节存放在内存的低地址端

<font color=red>ARM处理器使用的是小端模式;；网络字节数据用的大端模式</font>



![image-20200811105053657](D:\Saber_Workshop\Personal\Doc\Markdown_Note\Book\协议栈设计_LwIP笔记.assets\image-20200811105053657.png)





#### ARP(地址解析协议)表

动态映射：将32位的IP地址转换为对应48位的MAC地址 

核心：ARP缓存表；对缓存表的建立、更新、查询等操作

```c
struct etharp_entry{	// 缓存表项
    #if ARP_QUEUEING
    	struct etharp_q_entry *q;	//数据包缓冲队列指针
    #endif 
    	struct ip_addr ipaddr;		// 目标IP地址
    	struct eth_addr	ethaddr;	// MAC地址
    	enum etharp_state state;	// 描述该entry的状态
    	u8_t ctime;				   // 描述entry的时间信息。当大于规定的值时，该表项被内核删除
    	struct netif *netif;		// 相应网络接口信息
};

enum etharp_state{
    ETHARP_STATE_EMPTY = 0,
    ETHARP_STATE_PENDING,	// 处于不稳定状态，会发送一个广播ARP请求到数据链路上，让对应IP地址的主机回应其MAC地址
    ETHARP_STATE_STABLE	
};

static struct etharp_entry arp_table[ARP_TABLE_SIZE];		//数组方式创建ARP缓存表
```

![image-20200812150615770](D:\Saber_Workshop\Personal\Doc\Markdown_Note\Book\协议栈设计_LwIP笔记.assets\image-20200812150615770.png)

两个字节长的以太网帧类型表示后面数据的类型。对于 ARP 请求或应答数据包来说，
该字段的值为 0x0806，对于 IP 数据包来说，该字段的值为 0x0800。  

![image-20200812150904760](D:\Saber_Workshop\Personal\Doc\Markdown_Note\Book\协议栈设计_LwIP笔记.assets\image-20200812150904760.png)



![image-20200812151036425](D:\Saber_Workshop\Personal\Doc\Markdown_Note\Book\协议栈设计_LwIP笔记.assets\image-20200812151036425.png)



```c
// 数据报头
struct etharp_hdr{
    PACK_STRUCT_FIELD(struct eth_hdr ethhdr); // 14 字节的以太网数据报头
    PACK_STRUCT_FIELD(u16_t hwtype); // 2 字节的硬件类型
    PACK_STRUCT_FIELD(u16_t proto); // 2 字节的协议类型
    PACK_STRUCT_FIELD(u16_t _hwlen_protolen); // 两个 1 字节的长度字段
    PACK_STRUCT_FIELD(u16_t opcode); // 2 字节的操作字段 op
    PACK_STRUCT_FIELD(struct eth_addr shwaddr); // 6 字节源 MAC 地址
    PACK_STRUCT_FIELD(struct ip_addr2 sipaddr); // 4 字节源 IP 地址
    PACK_STRUCT_FIELD(struct eth_addr dhwaddr); // 6 字节目的 MAC 地址
    PACK_STRUCT_FIELD(struct ip_addr2 dipaddr); // 4 字节目的 IP 地址
}PACK_STRUCT_STRUCT;
// PACK_STRUCT_FIELD()是防止编译器字对齐的宏定义
```



#### ARP表查询

ARP攻击，针对是针对以太网地址解析协议（ ARP）的一种攻击技术。在局域网中， ARP
病毒收到广播的 ARP 请求包，能够解析出其它节点的 (IP, MAC) 地址, 然后病毒伪装为目
的主机，告诉源主机一个假 MAC 地址，这样就使得源主机发送给目的主机的所有数据包都
被病毒软件截取，而源主机和目的主机却浑然不知。 ARP 攻击通过伪造 IP 地址和 MAC 地
址实现 ARP 欺骗，能够在网络中产生大量的 ARP 通信量使网络阻塞，攻击者只要持续不断
的发出伪造的 ARP 响应包就能更改目标主机 ARP 缓存中的 IP-MAC 条目。 ARP 协议在设
计时未考虑网络安全方面的特性，这就注定了其很容易遭受 ARP 攻击。黑客只要在局域网
内阅读送上门来的广播 ARP 请求数据包，就能偷听到网内所有的 (IP, MAC)地址。而源节
点收到 ARP 响应时，它也不会质疑，这样黑客很容易冒充他人。  

```c
 // 寻找一个匹配的ARP项或者创建一个新的ARP表项，并返回该表项的索引号
static s8_t find_entry(struct ip_addr *ipaddr, u8_t flags)
```

lwip 有一个比较巧妙的地方 ,LWIP 中有个全局的变量 etharp_cached_entry，它始终保存着上次用到的索引号，如
果这个索引恰好就是我们要找的内容，且索引的表项已经处于 stable 状态，那就直接返回这个索引号就完成了 。

```c
// 向给定的 IP 地址发送一个数据包或者发送一个ARP请求，
err_t etharp_query(struct netif *netif, struct ip_addr *ipaddr, struct pbuf *q)
```



```c
// 该函数用于更新 ARP 缓存表中的表项或者在缓存表中插入一个新的表项。
// 该函数会在收到一个 IP 数据包或 ARP 数据包后被调用。
static err_t update_arp_entry(struct netif *netif, struct ip_addr *ipaddr, struct eth_addr *ethaddr, u8_t flags)
```

<img src="D:\Saber_Workshop\Personal\Doc\Markdown_Note\Book\协议栈设计_LwIP笔记.assets\image-20200812171043047.png" alt="image-20200812171043047" style="zoom:80%;" />





#### IP层

IP 数据报头  

```c
struct ip_hdr {
    PACK_STRUCT_FIELD(u16_t _v_hl_tos); // 前三个字段：版本号、首部长度、服务类型
    PACK_STRUCT_FIELD(u16_t _len); // 总长度
    PACK_STRUCT_FIELD(u16_t _id); // 标识字段
    PACK_STRUCT_FIELD(u16_t _offset); // 3 位标志和 13 位片偏移字段
    #define IP_RF 0x8000 //
    #define IP_DF 0x4000 // 不分组标识位掩码
    #define IP_MF 0x2000 // 后续有分组到来标识位掩码
    #define IP_OFFMASK 0x1fff // 获取 13 位片偏移字段的掩码
    PACK_STRUCT_FIELD(u16_t _ttl_proto); // TTL 字段和协议字段
    PACK_STRUCT_FIELD(u16_t _chksum); // 首部校验和字段
    PACK_STRUCT_FIELD(struct ip_addr src); // 源 IP 地址
    PACK_STRUCT_FIELD(struct ip_addr dest); // 目的 IP 地址
} PACK_STRUCT_STRUCT;
```

