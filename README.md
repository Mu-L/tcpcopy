# [TCPCopy](https://github.com/session-replay-tools/tcpcopy) - A TCP Stream Replay Tool

TCPCopy is a TCP stream replay tool for realistic testing of Internet server applications. 


## Description
Although real live traffic is crucial for testing Internet server applications, accurately simulating it is challenging due to the complexity of online environments. To enable more realistic testing, TCPCopy was developed as a live flow reproduction tool that generates test workloads closely resembling production workloads. TCPCopy is widely used by companies in China.

TCPCopy minimally impacts the production system, consuming only additional CPU, memory, and bandwidth. The reproduced workload mirrors the production environment in terms of request diversity, network latency, and resource usage.

## Use Cases
* Distributed Stress Testing
  - Use TCPCopy to replicate real-world traffic for stress testing your server software, uncovering bugs that only appear under high-stress conditions.
* Live Testing
  - Validate the stability of new systems and identify bugs that only manifest in real-world scenarios
* Regression testing
  - Ensure that recent changes have not introduced new issues.
* Performance comparison
  - Compare system performance across different versions or configurations.


## Architecture 

![tcpcopy](https://github.com/wangbin579/auxiliary/blob/master/images/tcpcopy.png)

As shown in Figure 1, TCPCopy is composed of two components: *tcpcopy* and *intercept*. The *tcpcopy* component runs on the online server, capturing live requests, while *intercept* operates on the assistant server, performing tasks such as passing response information to *tcpcopy*. The test application itself runs on the target server.

By default, *tcpcopy* employs raw socket input to capture online packets at the network layer, handling essential processes like TCP interaction simulation, network latency control, and upper-layer interaction simulation. It then sends packets to the target server using raw socket output (illustrated by pink arrows in the figure).

The only required operation on the target server is configuring route commands to direct response packets (indicated by green arrows in the figure) to the assistant server.

The *intercept* component's role is to forward the response header (by default) to *tcpcopy*. It captures the response packets, extracts the response header information, and sends this information to *tcpcopy* via a dedicated channel (represented by purple arrows in the figure). Upon receiving the response header, *tcpcopy* uses the information to modify the attributes of online packets and proceeds to send subsequent packets. It is important to note that responses from the target server are routed to the assistant server, which functions as a black hole.


## Quick Start

For *intercept*, you have two options:

* [Download the latest intercept release](https://github.com/session-replay-tools/intercept/releases).
* Clone the repository:
  `git clone git://github.com/session-replay-tools/intercept.git`.

For *tcpcopy*, you also have two options
* [Download the latest tcpcopy release](https://github.com/session-replay-tools/tcpcopy/releases).
* Clone the repository:
  `git clone git://github.com/session-replay-tools/tcpcopy.git`.


## Installing intercept on the Assistant Server
1. Navigate to the *intercept* directory:<br>
   `cd intercept`
2. Run the configuration script:<br>
   `./configure` <br>
   Optionally, specify any necessary configuration options.
3. Compile the source code:<br>
   `make`
4. Install the *intercept* tool:<br>
   `make install`

### Configure Options for *intercept*
    --single            Run `intercept` in non-distributed mode.
    --with-pfring=PATH  Specify the path to the PF_RING library sources.
    --with-debug        Compile `intercept` with debug support, with logs saved to a file.


## Installing tcpcopy on the Online Server
1. Navigate to the *tcpcopy* directory: <br>
   `cd tcpcopy`
3. Run the configuration script: <br>
   `./configure` <br>
    Include any necessary configuration options as needed.
4. Compile the source code: <br>
   `make`
5. Install the *tcpcopy* tool: <br>
   `make install`


### Configure Options for *tcpcopy*
    --offline                   Replay TCP streams from a pcap file.
    --pcap-capture              Capture packets at the data link layer.
    --pcap-send                 Send packets at the data link layer instead of the IP layer.
    --with-pfring=PATH          Specify the path to the PF_RING library sources.
    --set-protocol-module=PATH  Set `tcpcopy` to work with an external protocol module.
    --single                    If both `intercept` and `tcpcopy` are configured with the --single option, only one `tcpcopy` instance will work with `intercept`, leading to better performance. 
    --with-tcmalloc             Use tcmalloc instead of malloc.
    --with-debug                Compile `tcpcopy` with debug support, with logs saved to a file.


   
## Running TCPCopy
Assume that both *tcpcopy* and *intercept* are configured using `./configure`.
 
### 1) On the Target Server Running Server Applications:
      Configure the route commands to direct response packets to the assistant server.
      For example, if 61.135.233.161 is the IP address of the assistant server, use the following route command to direct all responses from clients in the 62.135.200.x range to the assistant server:
           route add -net 62.135.200.0 netmask 255.255.255.0 gw 61.135.233.161

### 2) On the Assistant Server Running *intercept* (Root Privilege or CAP_NET_RAW Capability Required):

       ./intercept -F <filter> -i <device,>
       
       Note that the filter format is the same as the pcap filter.
       For example:
          ./intercept -i eth0 -F 'tcp and src port 8080' -d
          
	  In this example, `intercept` will capture response packets from a TCP-based application listening on port 8080, using the eth0 network device.
    
	
### 3) On the Online Source Server (Root Privilege or CAP_NET_RAW Capability Required):
       ./tcpcopy -x localServerPort-targetServerIP:targetServerPort -s <intercept server,> [-c <ip range,>]

       For example(assume 61.135.233.160 is the IP address of the target server):

       ./tcpcopy -x 80-61.135.233.160:8080 -s 61.135.233.161 -c 62.135.200.x
	
       In this example, `tcpcopy` captures packets on port 80 from the current server, changes the client IP address to one from the 62.135.200.x range, and sends these packets to port 8080 on the target server (61.135.233.160). It also connects to 61.135.233.161 to request intercept to forward response packets.
       While the -c parameter is optional, it is used here to simplify route commands.

## Note
1. Platform: Tested only on Linux (kernel 2.6 or above).
2. Packet Loss: TCPCopy may lose packets, which could result in lost requests.
3. Permissions: Requires root privilege or the CAP_NET_RAW capability (e.g., setcap CAP_NET_RAW=ep tcpcopy).
4. Connection Type: Currently supports only client-initiated connections.
5. SSL/TLS: Does not support replay for applications using SSL/TLS.
6. MySQL Session Replay: For details, visit [session-replay-tools](https://github.com/session-replay-tools/).
7. IP Forwarding: Ensure ip_forward is not enabled on the assistant server.
8. Help: For more information, run ./tcpcopy -h or ./intercept -h.

## Influential Factors
Several factors can impact TCPCopy, as detailed in the following sections.

### 1. Capture Interface
By default, *tcpcopy* uses a raw socket input interface to capture packets at the network layer on the online server. Under high load, the system kernel may drop some packets.
If configured with --pcap-capture, *tcpcopy* captures packets at the data link layer and can filter packets in the kernel. Using *PF_RING* with pcap capturing can reduce packet loss.
For optimal capture, consider mirroring ingress packets via a switch and distributing the traffic across multiple machines with a load balancer.


### 2. Sending Interface
*tcpcopy* defaults to using a raw socket output interface to send packets at the network layer to the target server. To avoid ip_conntrack issues or improve performance, use `--pcap-send` to send packets at the data link layer instead.


### 3.On the Way to the Target Server 
Packets sent by *tcpcopy* may face challenges before reaching the target server. If the source IP address is the end-user's IP (by default), security devices may drop the packet as invalid or forged. To test this, use *tcpdump* on the target server. If packets are successfully sent within the same network segment but not across segments, packets may be dropped midway.

To address this, deploy *tcpcopy*, target applications, and intercept within the same network segment. Alternatively, use a proxy in the same segment to forward packets to the target server in another segment.

Deploying the target server’s application on a virtual machine within the same segment may still encounter these issues.

### 4. OS of the Target Server
The target server might use *rpfilter* to verify the legitimacy of source IP addresses, dropping packets deemed forged. If packets are captured by *tcpdump* but not processed, check *rpfilter* settings and adjust or remove them as needed. Other issues like iptables settings may also affect *tcpcopy*.


### 5. Applications on the Target Server
Applications on the target server may not process all requests promptly. Bugs or limitations in the application can lead to delayed responses or unprocessed requests in the socket buffer.

### 6. OS of the assistant Server
Ensure that *ip_forward* is set to false on the assistant server to prevent it from routing packets and ensure it functions as a black hole.

## Release History
+ 2014.09  v1.0    TCPCopy released
+ 2024.09  v1.0    Open source fully uses English

## Bugs and feature requests
Have a bug or a feature request? [Please open a new issue](https://github.com/session-replay-tools/tcpcopy/issues). Before opening any issue, please search for existing issues.

## Copyright and license

Copyright 2024 under [the BSD license](LICENSE).


