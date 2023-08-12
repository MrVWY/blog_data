---
title: "C language - TCP/IP .h file and use Xdp"
date: "2021-01-02 16:17:01"
categories:
  - C
tags:
  - C
toc: true
math: true
type: about
---

Commonly use .h network file

### IPv4 header

- `netinet/ip.h`：`struct ip`

  ```c
  struct ip {
  #if BYTE_ORDER == LITTLE_ENDIAN 
  	u_char	ip_hl:4,		/* header length */
  		ip_v:4;			/* version */
  #endif
  #if BYTE_ORDER == BIG_ENDIAN 
  	u_char	ip_v:4,			/* version */
  		ip_hl:4;		/* header length */
  #endif
  	u_char	ip_tos;			/* type of service */
  	short	ip_len;			/* total length */
  	u_short	ip_id;			/* identification */
  	short	ip_off;			/* fragment offset field */
  #define	IP_DF 0x4000			/* dont fragment flag */
  #define	IP_MF 0x2000			/* more fragments flag */
  	u_char	ip_ttl;			/* time to live */
  	u_char	ip_p;			/* protocol */
  	u_short	ip_sum;			/* checksum */
  	struct	in_addr ip_src,ip_dst;	/* source and dest address */
  };
  ```

- `linux/ip.h`：`struct iphdr `

  ```c
  struct iphdr {
      #if defined(__LITTLE_ENDIAN_BITFIELD)
          __u8    ihl:4,
                  version:4;
      #elif defined (__BIG_ENDIAN_BITFIELD)
          __u8    version:4,
                  ihl:4;
      #else
          #error  "Please fix <asm/byteorder.h>"
      #endif
           __u8   tos;
           __u16  tot_len;
           __u16  id;
           __u16  frag_off;
           __u8   ttl;
           __u8   protocol;
           __u16  check;
           __u32  saddr;
           __u32  daddr;
           /*The options start here. */
  };
  ```

  

### Tcp header

- `netinet/tcp.h` ： `struct tcphdr`

  ```c
  struct tcphdr {
  	u_short	th_sport;		/* source port */
  	u_short	th_dport;		/* destination port */
  	tcp_seq	th_seq;			/* sequence number */
  	tcp_seq	th_ack;			/* acknowledgement number */
  #if BYTE_ORDER == LITTLE_ENDIAN 
  	u_char	th_x2:4,		/* (unused) */
  		th_off:4;		/* data offset */
  #endif
  #if BYTE_ORDER == BIG_ENDIAN 
  	u_char	th_off:4,		/* data offset */
  		th_x2:4;		/* (unused) */
  #endif
  	u_char	th_flags;
  #define	TH_FIN	0x01
  #define	TH_SYN	0x02
  #define	TH_RST	0x04
  #define	TH_PUSH	0x08
  #define	TH_ACK	0x10
  #define	TH_URG	0x20
  	u_short	th_win;			/* window */
  	u_short	th_sum;			/* checksum */
  	u_short	th_urp;			/* urgent pointer */
  };
  ```

  

- `linux/tcp.h` ： `struct tcphdr`

  

### Udp header

- `netinet/udp.h`： `struct udphdr`

  ```c
  struct udphdr
  {
    u_int16_t uh_sport;                /* source port */
    u_int16_t uh_dport;                /* destination port */
    u_int16_t uh_ulen;                /* udp length */
    u_int16_t uh_sum;                /* udp checksum */
  };
  ```

  

- `linux/udp.h`： `struct udphdr`

### sk_buff

- `linux/skbuff.h`：`struck sk_buff`

  ```c
  struct sk_buff {
  	/* These two members must be first. */
  	struct sk_buff		*next;
  	struct sk_buff		*prev;
  
  	struct sock		*sk;
  	struct skb_timeval	tstamp;
  	struct net_device	*dev;
  	struct net_device	*input_dev;
  	//transport 
  	union {
  		struct tcphdr	*th;
  		struct udphdr	*uh;
  		struct icmphdr	*icmph;
  		struct igmphdr	*igmph;
  		struct iphdr	*ipiph;
  		struct ipv6hdr	*ipv6h;
  		unsigned char	*raw;
  	} h;
  	//network
  	union {
  		struct iphdr	*iph;
  		struct ipv6hdr	*ipv6h;
  		struct arphdr	*arph;
  		unsigned char	*raw;
  	} nh;
  	//mac
  	union {
  	  	unsigned char 	*raw;
  	} mac;
  	//route
  	struct  dst_entry	*dst;
  	struct	sec_path	*sp;
    .........
  };
  ```

  

### Icmp header

- `netinet/ip_icmp.h`： `struct icmphdr`

- `linux/icmp.h`： `struct icmphdr`

- type and code 

  reference：https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol

  

### Arp header

- `net/if_arp.h`：   `struct arphdr` 
- `linux/if_arp.h` ：`struct arphdr`



### Ethernet header

- `netinet/if_ether.h`：
- `linux/if_ether.h` ： `struct ethhdr`



### Use  iovisor/Bcc

- `linux/bpf.h`：

  ```c
  enum xdp_action {
  	XDP_ABORTED = 0,
  	XDP_DROP,
  	XDP_PASS,
  	XDP_TX,
  	XDP_REDIRECT,
  };
  
  struct xdp_md {
  	__u32 data;
  	__u32 data_end;
  	__u32 data_meta;
  	/* Below access go through struct xdp_rxq_info */
  	__u32 ingress_ifindex; /* rxq->dev->ifindex */
  	__u32 rx_queue_index;  /* rxq->queue_index  */
  
  	__u32 egress_ifindex;  /* txq->dev->ifindex */
  };
  ```

- example

  - filter.c


  ```c
  #include <linux/bpf.h>
  #include <linux/if_ether.h>
  #include <linux/ip.h>
  #include <linux/in.h>
  #include <linux/udp.h>
  
  int filter(struct xdp_md *ctx) {
      void *data = (void *)(long)ctx->data;
      void *data_end = (void *)(long)ctx->data_end;
      struct ethhdr *eth = data; //指向数据包开头
      if ((void*)eth + sizeof(*eth) <= data_end) {
      	struct iphhdr *ip = data + sizeof(*eth); //指向IP
          if ((void*)ip + sizeof(*ip) <= data_end) {
              ......
          }
      }
      return XDP_PASS;
  }
  ```

   run.py

  ```python
  from bcc import BPF
  device = "lo"
  b = BPF(src_file="filter.c")
  f = b.load_func("filter", BPF.XDP)
  b.attach_xdp(device, f, 0)
  try:
      b.trace_print()
  except ... :
      ....
      
  b.remove_xdp(device, 0)
  ```

  

- Bcc  Related Reference 

  [What exactly is usage of cursor_advance in BPF? ](https://stackoverflow.com/questions/60249948/what-exactly-is-usage-of-cursor-advance-in-bpf)

### Reference Guide

https://github.com/iovisor/bcc/tree/7e3f0c08c7c28757711c0a173b5bd7d9a31cf7ee/examples

### Function  Guide

https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md