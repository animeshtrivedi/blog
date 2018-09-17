# Investigating the RoCE (v2) protocol 

## Index 
1. [Introduction](#introduction)
2. [Measuring RDMA traffic on Mellanox NICs](#measuring-rdma-traffic-on-mellanox-nics)
3. [Reasoning about the RDMA traffic](#reasoning-about-the-rdma-traffic)
    - 3.1 [One outstanding request](#one-outstanding-request)
    - 3.2 [Multiple outstanding requests](#multiple-outstanding-requests)
    - 3.3 [Bandwidth tests](#bandwidth-tests)
4. [Deciphering RoCE Header](#deciphering-roce-header)
5. [Conclusion](#conclusion)

## Introduction 
Benchmarking distributed systems is a hard job. I often need to ensure that the performance that 
I am measuring in my benchmark is sensible, and hence I cross-check the performance with a 
variety of independent tools. 


Network benchmarking on on iWARP (RDMA over IP/TCP) use to be (slightly) easy because on our Chelsio 
iWARP RDMA NICs, RDMA packets are accounted as any packet (offloaded or not) and would show up in 
the counters of the NIC device in the kernel. So when one does `ifconfig`, the number of packet 
transmitted, received, or number of bytes transmitted or received would show RDMA traffic as well. 
But our new 100 Gbps Mellanox cards account offloaded and non-offloaded traffic differently. So the 
first thing I need to find out is how to measure RDMA traffic on the Mellanox NICs? 

## Measuring RDMA traffic on Mellanox NICs 

Mellanox has a good write-up where to find counters for their RDMA traffic. It is available at 
https://community.mellanox.com/docs/DOC-2532. The document shows insane (!) amount of counters,
as they allow unprecedented access into the working of a RNIC (more on this later). Today we 
are interested in benign RDMA counters which are classified under `vPORT` counters. The 
interesting bit of counters are: 

| Counter       |  Description|
| :------------- |:-------------|
| rx_vport_rdma_unicast_packets	 | RDMA unicast packets received |  
| rx_vport_rdma_unicast_bytes  | RDMA unicast bytes received |
| tx_vport_rdma_unicast_packets  | RDMA unicast packets transmitted |
| tx_vport_rdma_unicast_bytes | RDMA unicast bytes transmitted |

So, here is a [nice bash script](https://github.com/animeshtrivedi/utilities/blob/master/network/rnic-counter.sh) 
to do the work for you and show per-second RDMA traffic on your network card, something like: 

```bash
$ ./rnic-counter.sh NIC 
TX(bytes)   RX(bytes)  | TX(pkts)   RX(pkts)   
29813258    39430438   | 480859     480858     
29821194    39440934   | 480988     480987     
29842646    39469306   | 481333     481333     
29900802    39546222   | 482270     482271     
29831362    39454464   | 481151     481151     
29819210    39438392   | 480956     480955     
29885798    39526378   | 482028     482028     
29908676    39556554   | 482397     482397     
29890758    39532938   | 482110     482109     
29895408    39539088   | 482185     482185     
^C
```

## Reasoning about the RDMA traffic 

As a next step, I want to understand how these counters correlate to the application-level performance. 
For example, if I see that my distributed hash-table look-up for 8 bytes is delivering 100M operations/sec,
does this performance show up as 100M packets/sec on the wire? or 800M bytes/sec bandwidth? I use the 
term *packet* for anything that an RNIC transfers or receives, and *operation* which is a application/benchmark-level logical operation. For example, a 4MB RDMA read is one operation for an application that might 
result in 1000s of packets on the wire. 

### One outstanding operation 

We analyze a simple RDMA write operation where the traffic flows in one direction (and ack
and credits in the other). For a single outstanding 1-byte write request benchmark, 
I get around 478K operations per second. While running the benchmark, the network profile 
looks like (all network profiles are done on the server side, which for WRITE is receiving 
the WRITE traffic):
```bash
TX(bytes)   RX(bytes)  | TX(pkts)   RX(pkts)   
29857464    37562538   | 481571     481572     
29914318    37634064   | 482489     482489     
29901980    37618620   | 482290     482290
[...]
```
So, if you see the RX packet rate that is pretty close to the RDMA write operation rate. That make 
sense. There are RDMA write packet flowing in one direction and ack/credit packets going in the 
other. But the bandwidth, seems odd. For 478K/sec, 1 byte operations, that bandwidth should be 
around 478KB/sec. But the network counters show close to 29 MB (!). We will come back to this 
later when we analyze the RoCE headers. Atleast for now, for a single outstanding requests the 
packet rate closely follow the RDMA operation rate. So, looking the the network profile like 
above, I can guess that the RDMA application should be getting close to ~480K operations/sec. 
(The analysis looks the same for RDMA READ operations.)

Lets have another look send/recv benchmark where in an RPC request/response style communication 
pattern, operations are issued in **both** directions. Here is how a 1-byte request-response 
traffic looks like: 

```bash
TX(bytes)   RX(bytes)  | TX(pkts)   RX(pkts)   
28722802    29649408   | 463272     463272     
28717160    29643586   | 463181     463182
28865464    29796674   | 465573     465574     
28836882    29767040   | 465110     465110     
[...]        
```
So, with ~465K packets/sec, this benchmark is sending sends and ACKs (for receives). So we 
should half the rate to 232.5K/sec - this is our expected performance rate. And voilà, 
the benchmark reports 231K operations/sec. 

### Multiple outstanding requests 

Now, I am going to issue 8 outstanding 1-byte write requests, and keep posting more as soon 
as they finish. So the idea is to always have close to 8 requests in flight. Here is how the 
network profile looks like: 
```bash
TX(bytes)   RX(bytes)  | TX(pkts)   RX(pkts)   
191355746   245078808  | 3086381    3142035    
191077924   244886226  | 3081900    3139566    
191334046   244972338  | 3086033    3140670
[...]
```
Can you guess what is the application-level performance? Probably 3.1M operations/sec? as that 
is the packet rate - and it is true. The write performance (for a unidirectional WRITE traffic)
is 3.1M operations/sec.  

It gets more interesting for bi-directional send/recv. So, here is a profile for a 1 byte send/recv 
benchmark with 8 in pipeline. 
```bash
TX(bytes)   RX(bytes)  | TX(pkts)   RX(pkts)   
150260348   149876010  | 2423553    2417351    
149851644   148401464  | 2416960    2393567    
153220290   152694034  | 2471295    2462804    
153655406   152399782  | 2478309    2458061   
[...]
```
So, probably the throughput is around 2.4M/2 = 1.2M operations/sec? But in reality is it is 1.4M ops/sec, 
it is close but not exactly. If I push 32 in flight instead of 8, then 
```bash
TX(bytes)   RX(bytes)  | TX(pkts)   RX(pkts)   
379416688   379564124  | 6119620    6122003    
380508012   377267272  | 6137219    6084950    
378343468   378792224  | 6102315    6109562
[...]
```
So, 6.1M/2 = 3.1M operations/sec? The benchmark level performance is 4M operations/sec. For 64 in-flight,  
the packet rate is close to 10M packets/sec, but the benchmark performance is 6.1M operations/sec. You  
see the pattern. 
 
The reason for this discrepancy is that in bi-directional traffic - the credit/ack packet can be piggy 
backed on the outgoing data packet. So, remember when there was only 1 outstanding request, there is 
exactly one response coming back from the server. But for low latency reasons, the RNIC cannot wait 
indefinitely and hence must send a credit/ack packet back without waiting for the outgoing response. 
Hence,  we saw 2x number of packets on the wire. But as we increase the number of in-flight packets, 
the probability of finding an outgoing data packet (on which the NIC can piggy-back ACK) increases. 
So in theory the packet rate can be anywhere in between [ operationRate , 2 x operationRate]. 
And it become more difficult to predict with accuracy what is the operation rate for a given packet 
rate. 

### Bandwidth tests 

Lets have a quick look at the bandwidth patterns. They look for sensible, for a reasonable size (we 
discuss this in the next section), the bandwidth matches 1-1 with what the RNIC reports. For example, 
here is the stream for 1MB RDMA WRTIE transfer with 8 in flight. 
```bash
TX(bytes)   RX(bytes)   | TX(pkts)   RX(pkts)   
6534490     12487635292 | 105394     3006125    
6551416     12480889084 | 105669     3004502    
6545340     12486214592 | 105570     3005784    
6537900     12491394710 | 105450     3007031    
[...]
```
So the server is receiving around 1.24 GB/sec, which is what we expect on a 100 Gbps link. And the 
 application reports 12.3 GB/sec. 

For small packets, it looks very different. Here is the same experiment, 8 in-flight, 1 byte WRITEs,
 ```bash
TX(bytes)   RX(bytes)  | TX(pkts)   RX(pkts)   
191029130   244239606  | 3081115    3131278    
190989450   244257390  | 3080474    3131505    
190926458   244162230  | 3079459    3130284    
190786710   243929244  | 3077205    3127299    
[...] 
 ```
So the packet rate matches the operation rate, which is 3.1M operations/sec. But the bandwidth? The RNIC
reports that it is transmitting 243 MB/sec, whereas with 3.1M 1-byte operations we would expect 3.1 MB/sec. 
So what is going on? So here the protocol overheads and packet size comes into the play what we will 
discuss in the next section. 

## Deciphering RoCE Header 

So it we divide, the bytes (243929244) with the number of packets (3127299), we get 77.9 or 78 bytes/packet.
For every 1 byte transfer, RoCE actually transmits 78 bytes. So, lets verify this. Lets make zero length
write requests. Here is the profile for that: 
```bash
TX(bytes)   RX(bytes)  | TX(pkts)   RX(pkts)   
259787564   325471240  | 4190122    4398260  ==> (rx) 74 bytes/packet    
260477438   326227298  | 4201248    4408476  ==>  (rx) 74 bytes/packet
```
So for a zero length payload WRITE operation, RoCE is transmitting 74 bytes. The read has the similar pattern.  
This can be seen in more detail, if we have a look at the RoCE header which is shown at 
https://community.mellanox.com/docs/DOC-1451 as 

![RoCE v2 header](https://community.mellanox.com/servlet/JiveServlet/showImage/102-1451-29-84647/RoCE+frame+Format.png)

So, here you can see that RoCE header sum in total to 70 bytes. There is still missing 4 bytes, as we expected this
size to be of 74 bytes. I don't know where that comes from. Anyone? 

1 byte payload is backed as a 4 byte (to make it naturally aligned), and payload is incremented in the 
step of 4 bytes. So, a zero byte packet size is 74 bytes, 1-4 bytes packet size is 78 bytes, and 5-8 
byte packet size is 82 bytes.
   
   
So from here we can also calculate the efficiency of the RoCE network. It uses IB MTU of 4096 bytes. So 
if we calculate the [Ethernet overheads](https://en.wikipedia.org/wiki/Ethernet_frame#Structure) as : 42 bytes (= 7 bytes (Preamble) + 1 byte (delimiter) + 
12 bytes (src+dst) + 2 bytes(Ethertype) + 4 bytes (VLAN) + 4 bytes (CRC) + 12 bytes (Interframe Gap, 
YES this is also transmitted)). Then higher-level protocols include 20 bytes (IP) + 8 bytes (UDP) + 20 bytes 
(IB + checksum) = 48 bytes. So for a given MTU size of 4096, we get 
```concept
(4096 - 48) / (4096 + 42) = 0.978 or 97.8% efficiency
```
With 97.8% efficiency, one cannot get more than 97.8 Gbps out of a 100 Gbps link. 

For bandwidth tests when the payload size becomes larger than the protocols overheads (~90 bytes), the
bandwidth measurements will reflect the actual application performance. Any thing less than that, be careful 
what you see on the wire. 

The ACK packet size can be calcuated looking at the 1-byte TX WRITE traffic on the server side. It calculates to 62 bytes. 
I do not have a break down of how the ACK packet looks like. 

## Conclusion  

That is it folks. In conclusion: 
  - for a uni-directional traffic (WRITE, READ): operation and packet rates match 1:1. For reasonably large 
  request size, the bandwidth also matches 1:1. 
  - for bi-drectional send/recv traffic: if you have 1 outstanding then operation and packet rate matches 1:2. 
  For anything else, it can vary between the range of 1 - 2x. 
  - RoCE v2 has a large header, so be careful when you measure small message bandwidths and efficiency. 
  - There is mysterious 4 bytes in the 0-length payload RoCE header, I don't know where it comes from. If you know, let me know :) 
  - The RoCE v2 ACK packet is size is ~62 bytes, but I do not know how the packet looks like. 

Thanks Jonas Pfefferle for sitting with me to investigate this. 
