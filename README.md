# **OpenShift Latency & Bandwidth Testing for Edge**

<p align="center">
<img align="center" src="bandwidth.jpg" style="width: 800px;" border=0/>
</p>

Just recently a customer approached me with the question around bandwidth and latency in regards to installing a Single Node OpenShift cluster at edge sites.   They were concerned given the sites limited bandwidth (1Mbit) and latency (90ms) if the installer would time out before completing the installation.  This line of questioning started me thinking on how we could prove the answer with empirical evidence without having the customer actually go through the pain of doing a proof of concept.

The initial idea was to build a lab harness that would simulate the latency and bandwidth limitation of the site without actually having to be at the site.  To do this I figured I could take a a regular RHEL node that I would use as a KVM hypervisor and then create an empty virtual machine in the hypervisor that would eventually become my Single Node OpenShift node.  Since I had it lying around I just used a plain old Intel NUC (NUC8i7BEH).  The NUC has but one interface (eno1) on it but that really was all I needed for this scenario.   I decided to go with the networking diagram below:

<p align="center">
<img align="center" src="lab.jpeg" style="width: 800px;" border=0/>
</p>

We can see from the picture our baremetal network comes in on eno1 and then is attached to br10.  Then we take br10 and attach it to the virtual machine where it becomes enp1s0 inside the virtual machine.  Once OpenShift gets installed there will then be a br-ex interface on top of the enp1s0 interface.  This setup ensures I have a way to limit the bandwidth and latency from the start of the installation. This also allows me to use Traffic control (tc) which is a very useful Linux utility that gives you the ability to configure the kernel packet scheduler. The tool allows one to simulate packet delay and loss for UDP or TCP applications, or limit the bandwidth usage of a particular service to simulate Internet connections (DSL, Cable, T1, etc). I use tc to apply rules to the eno1 interface and then I can use iperf to actually show those rules having an impact on the network performance.   

Before we apply our actual requirement lets take a look at a few examples the first being our baseline on eno1 without any rules applied.  Here we will simply run the iperf3 client and point it at our running iperf server I have on another host:

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

From the iperf output above we can see we are getting standard networking from the eno1 interface which is just a plain 1GB ethernet interface.  Now lets apply a tc rule to set the rate to 512kbit and 100ms of latency and run iperf:

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

From the test results we can see the tc rule reduced the speed on the interface.  Now lets try again with a 256kbit rate setting:

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

Again we see further reduction in the performance of the interface which confirms that tc is doing what it should.  Finally lets set the interface for a more extreme rate of 128kbit which is the rate I want to test in my deployment.   

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

We can see from the iperf3 testing above the rate is a lot slower then our original eno1 interface baseline rate but that is okay because the idea here is to see what will happen when we go to deploy OpenShift on a slow link.   Further this test is much slower then the customers required 1Mbit link but again if it works here I don't see it failing on the faster link.

At this point I went to 


~~~bash
# wget -O discovery_image_sno3.iso 'https://api.openshift.com/api/assisted-images/images/af8a514b-68f1-4a55-95ef-8fe82be48445?arch=x86_64&image_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTc5MTYxNTIsInN1YiI6ImFmOGE1MTRiLTY4ZjEtNGE1NS05NWVmLThmZTgyYmU0ODQ0NSJ9.AjTpZOn1utQQU7nqc-wk5pYDvllbKiBhcLS31pAdlWE&type=full-iso&version=4.10'
--2022-07-15 11:16:05--  https://api.openshift.com/api/assisted-images/images/af8a514b-68f1-4a55-95ef-8fe82be48445?arch=x86_64&image_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTc5MTYxNTIsInN1YiI6ImFmOGE1MTRiLTY4ZjEtNGE1NS05NWVmLThmZTgyYmU0ODQ0NSJ9.AjTpZOn1utQQU7nqc-wk5pYDvllbKiBhcLS31pAdlWE&type=full-iso&version=4.10
Resolving api.openshift.com (api.openshift.com)... 54.145.108.20, 34.236.218.142
Connecting to api.openshift.com (api.openshift.com)|54.145.108.20|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1047527424 (999M) [application/octet-stream]
Saving to: ‘discovery_image_sno3.iso’

discovery_image_sno3.iso                             100%[=====================================================================================================================>] 999.00M   667KB/s    in 25m 33s 

2022-07-15 11:41:39 (667 KB/s) - ‘discovery_image_sno3.iso’ saved [1047527424/1047527424]
~~~

<p align="center">
<img align="center" src="sno3-low-link.png" style="width: 900px;" border=0/>
</p>
