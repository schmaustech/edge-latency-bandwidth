# **OpenShift Latency & Bandwidth Testing for Edge**

<p align="center">
<img align="center" src="pipe.jpg" style="width: 1000px;" border=0/>
</p>

Just recently a customer approached me with the question around bandwidth and latency in regards to installing a Single Node OpenShift cluster at edge sites.   They were concerned given the sites limited bandwidth (1Mbit) and latency (90ms) if the installer would time out before completing the installation.  This line of questioning started me thinking on how we could prove the answer with empirical evidence without having the customer actually go through the pain of doing a proof of concept.

The initial idea was to build a lab harness that would simulate the latency and bandwidth limitation of the site without actually having to be at the site.  To do this I figured I could take a a regular RedHat Enterprise Linux 8 node that I would use as a KVM hypervisor and then create an empty virtual machine that would eventually become my Single Node OpenShift node.  I decided to use a plain old Intel NUC (NUC8i7BEH) since I had it available.  The NUC has but one interface (eno1) on it but that really was all I needed for this scenario.   I then decided to configure networking like the networking diagram below:

<p align="center">
<img align="center" src="lab.jpeg" style="width: 800px;" border=0/>
</p>

We can see from the picture our baremetal network comes in on eno1 and then is attached to br10.  Then we take br10 and attach it to the virtual machine where it becomes enp1s0 inside the virtual machine.  Once OpenShift using OVN gets installed there will then be a br-ex interface on top of the enp1s0 interface.  This setup ensures I have a way to limit the bandwidth and latency from the start of the installation. This also allows me to use Traffic Control (tc) which is a very useful Linux utility that gives you the ability to configure the kernel packet scheduler. The tool allows one to simulate packet delay and loss for UDP or TCP applications, or limit the bandwidth usage of a particular service to simulate Internet connections (DSL, Cable, T1, etc). I will use tc to limit bandwidth and introduce delay for everything passing through the eno1 interfac.  Then I can use iperf to actually show those rules having an impact on the network performance.   

Before we apply our actual requirement lets take a look at a few examples the first being our baseline on eno1 without any rules applied.  Here we will simply run the iperf client and point it at our running iperf server I have on another host at ipaddress 192.168.0.5:

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

From the test results we can see the tc rule reduced the speed on the eno1 interface.  Now lets try again with a 256kbit rate setting:

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

Again we see further reduction in the performance of the interface which confirms that tc is doing what it should.  Finally lets set the eno1 interface for a more extreme rate of 128kbit which is the rate I want to test in my deployment.   

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

We can see from the iperf testing above the rate is a lot slower then our original eno1 interface baseline rate but that is okay because the idea here is to see what will happen when we go to deploy OpenShift on a slow link.   Further this test is much slower then the customers required 1Mbit link but again if it works here I don't see it failing on the 1Mbit link.

I should also point out that if one wants to see what the current tc rules applied to a host are they can use the following syntax:

~~~bash
#  tc -s qdisc
qdisc noqueue 0: dev lo root refcnt 2 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
qdisc tbf 8001: dev eno1 root refcnt 2 rate 128Kbit burst 256Kb lat 100ms 
 Sent 636433273 bytes 7660842 pkt (dropped 322679, overlimits 19715820 requeues 9) 
 backlog 0b 0p requeues 9
qdisc noqueue 0: dev br30 root refcnt 2 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
qdisc noqueue 0: dev br20 root refcnt 2 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
qdisc noqueue 0: dev eno1.20 root refcnt 2 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
qdisc noqueue 0: dev eno1.30 root refcnt 2 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
qdisc noqueue 0: dev br10 root refcnt 2 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
qdisc noqueue 0: dev eno1.10 root refcnt 2 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
qdisc noqueue 0: dev virbr0 root refcnt 2 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
qdisc noqueue 8003: dev vnet1 root refcnt 2 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
~~~

We can see from the output above all the interfaces on this NUC are show but only the eno1 has actual bandwidth and latency rules applied which is exactly what we wanted.

At this point the framework of my lab is configured.  Now we need to get a Single Node OpenShift cluster deployed and for that I will use Assisted Installer.  For more information on using the Assisted Installer please see this reference [blog](https://cloud.redhat.com/blog/how-to-use-the-openshift-assisted-installer).  For my installation I will choose to install a single node deployment and once I get to the point where I can add hosts I will download the full ISO for this installation.  Since my lab is using a virtual machine I will download the ISO directly onto the RHEL8 KVM hypervisor host which already has the tc rate limiting applied to the primary eno1 interface.   My download experience is below:

~~~bash
# wget -O discovery_image_sno3.iso 'https://api.openshift.com/api/assisted-images/images/af8a514b-68f1-4a65-95ef-8fe82be48445?arch=x86_64&image_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTc5MTYxNTIsInN1YiI6ImFmOGE1MTRiLTY4ZjEtNGE1NS05NWVmLThmZTgyYmU0ODQ0NSJ9.AjTpZOn1utQQU7nqc-wk5pYDvllbKiBhcLS31pAdlWE&type=full-iso&version=4.10'
--2022-07-15 11:16:05--  https://api.openshift.com/api/assisted-images/images/af8a514b-68f1-4a65-95ef-8fe82be48445?arch=x86_64&image_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTc5MTYxNTIsInN1YiI6ImFmOGE1MTRiLTY4ZjEtNGE1NS05NWVmLThmZTgyYmU0ODQ0NSJ9.AjTpZOn1utQQU7nqc-wk5pYDvllbKiBhcLS31pAdlWE&type=full-iso&version=4.10
Resolving api.openshift.com (api.openshift.com)... 54.145.108.20, 34.236.218.142
Connecting to api.openshift.com (api.openshift.com)|54.145.108.20|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1047527424 (999M) [application/octet-stream]
Saving to: ‘discovery_image_sno3.iso’

discovery_image_sno3.iso 100% [===============================================================>] 999.00M 667KB/s in 25m33s 

2022-07-15 11:41:39 (667 KB/s) - ‘discovery_image_sno3.iso’ saved [1047527424/1047527424]
~~~

We can see from the download experience that it took 25 minutes to download the full ISO image which is just 999mb.   Normally when I download this image without the tc rate limiting it only takes ~2 minutes.  This further shows that our rate limiting setup is working by design and has established a slower more latent link.

With the image downloaded to the KVM hypervisor I can map it to the virtual machine and power on the virtual machine.  Once the virtual machine reports back into Assisted Installer I can select it and complete the steps to start the installation.  Normally a Single Node OpenShift install like this take about an hour in my lab environment.  However that is without the tc rate limiting in place which I suspect will lead to a slower install time due to the images needed to be pulled in from Quay.io.

The installation finally completed as we can see from the screenshot below:

<p align="center">
<img align="center" src="sno3.png" style="width: 900px;" border=0/>
</p>

The completed installation confirmed two things for us.  First it confirmed that the installation will be a lot slower when using a smaller more latent bandwidth link.  But it also confirmed that on this slow link I could still successfully install my SNO cluster without issue which was what my original customer requested more details on.

Lets go ahead and confirm the cluster is operational by looking at the cluster operator output:

~~~bash
$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.10.20   True        False         False      14s     
baremetal                                  4.10.20   True        False         False      3h7m    
cloud-controller-manager                   4.10.20   True        False         False      3h7m    
cloud-credential                           4.10.20   True        False         False      3h26m   
cluster-autoscaler                         4.10.20   True        False         False      178m    
config-operator                            4.10.20   True        False         False      3h46m   
console                                    4.10.20   True        False         False      9s      
csi-snapshot-controller                    4.10.20   True        False         False      3h35m   
dns                                        4.10.20   True        False         False      3h25m   
etcd                                       4.10.20   True        False         False      3h36m   
image-registry                             4.10.20   True        False         False      174m    
ingress                                    4.10.20   True        False         False      3h22m   
insights                                   4.10.20   True        False         False      52m     
kube-apiserver                             4.10.20   True        False         False      3h49m   
kube-controller-manager                    4.10.20   True        False         False      3h33m   
kube-scheduler                             4.10.20   True        False         False      3h39m   
kube-storage-version-migrator              4.10.20   True        False         False      3h40m   
machine-api                                4.10.20   True        False         False      144m    
machine-approver                           4.10.20   True        False         False      3h31m   
machine-config                             4.10.20   True        False         False      3h31m   
marketplace                                4.10.20   True        False         False      3h33m   
monitoring                                 4.10.20   True        False         False      156m    
network                                    4.10.20   True        False         False      3h44m   
node-tuning                                4.10.20   True        False         False      3h27m   
openshift-apiserver                        4.10.20   True        False         False      3h14m   
openshift-controller-manager               4.10.20   True        False         False      3h26m   
openshift-samples                          4.10.20   True        False         False      3h17m   
operator-lifecycle-manager                 4.10.20   True        False         False      154m    
operator-lifecycle-manager-catalog         4.10.20   True        False         False      155m    
operator-lifecycle-manager-packageserver   4.10.20   True        False         False      145m    
service-ca                                 4.10.20   True        False         False      3h42m   
storage                                    4.10.20   True        False         False      3h6m  
~~~

Indeed this cluster appears to be ready for its workloads and again provided an example of how to test link latency and bandwidth restrictions.  However before I go I want to point out one more thing regarding bandwidth and latency testing.

I was exploring my SNO master node from cli I wanted to see if the tc command was available on Red Hat CoreOS which is the underlying operating system of OpenShift:

~~~bash
[core@master-0 ~]$ hostname
master-0.sno3.schmaustech.com
[core@master-0 ~]$ which tc
/sbin/tc
[core@master-0 ~]$ tc 
Usage:	tc [ OPTIONS ] OBJECT { COMMAND | help }
	tc [-force] -batch filename
where  OBJECT := { qdisc | class | filter | chain |
		    action | monitor | exec }
       OPTIONS := { -V[ersion] | -s[tatistics] | -d[etails] | -r[aw] |
		    -o[neline] | -j[son] | -p[retty] | -c[olor]
		    -b[atch] [filename] | -n[etns] name | -N[umeric] |
		     -nm | -nam[es] | { -cf | -conf } path }
~~~

It indeed appears the tc command tool is available on RHCOS.  I bring this up because another popular latency/bandwidth request I see customers ask about is stretching the OpenShift cluster control plane across multiple datacenters where the latency/bandwidth is some value for their environment.   Now Red Hat does offer details and guidance on how to deal with stretched multi site clusters documented here in this [knowledge base article](https://access.redhat.com/articles/3220991).  However having tc available means one could test that configuration in a lab environment even if its in the same datacenter because one could rate limit and/or increase the latency between the control plane nodes.   Definitely a simpler approach then actually racking 3 nodes in 3 different data centers just for a POC scenario.  

In this blog we demonstrated how to potentially test a deployment for a Single Node OpenShift cluster when at an edge site with low bandwidth and high latency.  We also discuss the use of Traffic Control (tc) and how it can be used in this installation scenario and possibly along with stretch cluster scenarios.  Hopefully it provides some ideas for those interested in exploring these types of scenarios when it comes to OpenShift.
