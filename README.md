# **OpenShift Latency & Bandwidth Testing for Edge**

<img src="bandwidth.jpg" style="width: 1000px;" border=0/>

Just recently a customer approached me with the question around bandwidth and latency in regards to installing a Single Node OpenShift cluster at edge sites.   They were concerned given the sites limited bandwidth (1Mbit) and latency (90ms) if the installer would time out before completing the installation.  This line of questioning started me thinking on how we could prove the answer with empirical evidence without having the customer actually go through the pain of doing a proof of concept.

The initial idea was to build a lab harness that would simulate the latency and bandwidth limitation of the site without actually having to be at the site.  To do this I figured I could take a a regular RHEL node that I would use as a KVM hypervisor and then create an empty virtual machine in the hypervisor that would eventually become my Single Node OpenShift node.  Since I had it lying around I just used a plain old Intel NUC (NUC8i7BEH).  The NUC has but one interface (eno1) on it but that really was all I needed for this scenario.   I decided I would go ahead and build a bridge (br10) off the single interface that would then get attached to my virtual machine.  The networking diagram is below:





~~~bash
# iperf3 -c 192.168.0.5
Connecting to host 192.168.0.5, port 5201
[  5] local 192.168.0.4 port 34016 connected to 192.168.0.5 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   112 MBytes   939 Mbits/sec    0    382 KBytes       
[  5]   1.00-2.00   sec   111 MBytes   935 Mbits/sec    0    382 KBytes       
[  5]   2.00-3.00   sec   111 MBytes   930 Mbits/sec    0    382 KBytes       
[  5]   3.00-4.00   sec   112 MBytes   936 Mbits/sec    0    382 KBytes       
[  5]   4.00-5.00   sec   111 MBytes   928 Mbits/sec    0    382 KBytes       
[  5]   5.00-6.00   sec   111 MBytes   930 Mbits/sec    0    382 KBytes       
[  5]   6.00-7.00   sec   112 MBytes   936 Mbits/sec    0    382 KBytes       
[  5]   7.00-8.00   sec   111 MBytes   928 Mbits/sec    0    382 KBytes       
[  5]   8.00-9.00   sec   111 MBytes   932 Mbits/sec    0    382 KBytes       
[  5]   9.00-10.00  sec   112 MBytes   941 Mbits/sec    0    523 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  1.09 GBytes   934 Mbits/sec    0             sender
[  5]   0.00-10.04  sec  1.08 GBytes   928 Mbits/sec                  receiver

iperf Done.
~~~

~~~bash
# tc qdisc replace dev eno1 root tbf rate 512kbit buffer 256kb latency 100ms
# iperf3 -c 192.168.0.5
Connecting to host 192.168.0.5, port 5201
[  5] local 192.168.0.4 port 34020 connected to 192.168.0.5 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   509 KBytes  4.16 Mbits/sec    1   41.0 KBytes       
[  5]   1.00-2.00   sec   127 KBytes  1.04 Mbits/sec    0   41.0 KBytes       
[  5]   2.00-3.00   sec  0.00 Bytes  0.00 bits/sec    0   41.0 KBytes       
[  5]   3.00-4.00   sec  0.00 Bytes  0.00 bits/sec    0   41.0 KBytes       
[  5]   4.00-5.00   sec   127 KBytes  1.04 Mbits/sec    0   41.0 KBytes       
[  5]   5.00-6.00   sec  0.00 Bytes  0.00 bits/sec    0   41.0 KBytes       
[  5]   6.00-7.00   sec   127 KBytes  1.04 Mbits/sec    0   41.0 KBytes       
[  5]   7.00-8.00   sec  0.00 Bytes  0.00 bits/sec    0   41.0 KBytes       
[  5]   8.00-9.00   sec   127 KBytes  1.04 Mbits/sec    0   41.0 KBytes       
[  5]   9.00-10.00  sec  0.00 Bytes  0.00 bits/sec    0   41.0 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  1018 KBytes   834 Kbits/sec    1             sender
[  5]   0.00-10.13  sec   844 KBytes   683 Kbits/sec                  receiver

iperf Done.
~~~

~~~bash
# tc qdisc replace dev eno1 root tbf rate 256kbit buffer 256kb latency 100ms
# iperf3 -c 192.168.0.5
Connecting to host 192.168.0.5, port 5201
[  5] local 192.168.0.4 port 34024 connected to 192.168.0.5 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   443 KBytes  3.62 Mbits/sec    1   38.2 KBytes       
[  5]   1.00-2.00   sec   127 KBytes  1.04 Mbits/sec    0   38.2 KBytes       
[  5]   2.00-3.00   sec  0.00 Bytes  0.00 bits/sec    0   38.2 KBytes       
[  5]   3.00-4.00   sec  0.00 Bytes  0.00 bits/sec    0   38.2 KBytes       
[  5]   4.00-5.00   sec  0.00 Bytes  0.00 bits/sec    0   38.2 KBytes       
[  5]   5.00-6.00   sec  0.00 Bytes  0.00 bits/sec    0   38.2 KBytes       
[  5]   6.00-7.00   sec   127 KBytes  1.04 Mbits/sec    0   38.2 KBytes       
[  5]   7.00-8.00   sec  0.00 Bytes  0.00 bits/sec    0   38.2 KBytes       
[  5]   8.00-9.00   sec  0.00 Bytes  0.00 bits/sec    0   38.2 KBytes       
[  5]   9.00-10.00  sec  0.00 Bytes  0.00 bits/sec    0   38.2 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   697 KBytes   571 Kbits/sec    1             sender
[  5]   0.00-10.15  sec   543 KBytes   438 Kbits/sec                  receiver

iperf Done.
~~~



~~~bash
# tc qdisc replace dev eno1 root tbf rate 128kbit buffer 256kb latency 100ms
# iperf3 -c 192.168.0.5
Connecting to host 192.168.0.5, port 5201
[  5] local 192.168.0.4 port 34060 connected to 192.168.0.5 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   407 KBytes  3.33 Mbits/sec    0   1.41 KBytes       
[  5]   1.00-2.00   sec  0.00 Bytes  0.00 bits/sec    0   33.9 KBytes       
[  5]   2.00-3.00   sec   127 KBytes  1.04 Mbits/sec    0   33.9 KBytes       
[  5]   3.00-4.00   sec  0.00 Bytes  0.00 bits/sec    0   33.9 KBytes       
[  5]   4.00-5.00   sec  0.00 Bytes  0.00 bits/sec    0   33.9 KBytes       
[  5]   5.00-6.00   sec  0.00 Bytes  0.00 bits/sec    0   33.9 KBytes       
[  5]   6.00-7.00   sec  0.00 Bytes  0.00 bits/sec    0   33.9 KBytes       
[  5]   7.00-8.00   sec  0.00 Bytes  0.00 bits/sec    0   33.9 KBytes       
[  5]   8.00-9.00   sec  0.00 Bytes  0.00 bits/sec    0   33.9 KBytes       
[  5]   9.00-10.00  sec  0.00 Bytes  0.00 bits/sec    0   33.9 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   535 KBytes   438 Kbits/sec    0             sender
[  5]   0.00-10.39  sec   395 KBytes   311 Kbits/sec                  receiver

iperf Done.
~~~
