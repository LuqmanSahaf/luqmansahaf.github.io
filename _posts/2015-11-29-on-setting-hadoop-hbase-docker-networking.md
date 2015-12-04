---
layout: post
title: "On Setting Hadoop, HBase with Docker: Networking"
excerpt_separator: <!--excerpt_end-->
---

_I will explain how I set up networking for Hadoop and HBase clusters using Docker containers on more than one machines._

<!--excerpt_end-->

I have been developing an application for deployment at my [work](http://www.platalytics.com). This deployment application generally can deploy any service which has a Docker image available for it. Particularly, our uses involve services, relating Big Data Analytics, such as Hadoop HDFS, MapReduce, HBase, Kafka, Spark and many more.

I must say that there are many articles on how to do networking for Docker containers which span on more than one machine. However, the purpose of this writing is to share and discuss the problems we have faced during the set up, and the design choices we made.

### Network Namespaces

You should have basic understanding of how containers communicate on a single machine. For that, let me introduce the concept of network _namespace_.

> A network [namespace](http://man7.org/linux/man-pages/man8/ip-netns.8.html) is logically another copy of the network stack,
with its own routes, firewall rules, and network devices.

To get a good grasp of namespaces, do read [this](http://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/) and [this](http://containerops.org/2013/11/19/lxc-networking/).

Docker (or lxc) containers have their own namespaces. These namespaces are connected to the default namespace via a _Virtual Ethernet_ (or _veth_), which consists of two virtual _peer devices_. One peer device is assigned to the container and the other to the _bridge_. This acts like a pipe, through which the traffic passes to/from particular containers. In this way, the container can contact the outside world. There are [_masquerade_](http://en.wikipedia.org/wiki/Network_address_translation) rules defined in IP Tables so that to the outer world it appears that the device speaking is actually the host on which the container is running.

One benefit of the namespaces is that the container's network environment gets isolated, which is the main feature of containerization.

### Machines += 1

So, what happens when we add another machine to the network and try to start Hadoop services (NameNode and DataNode) on different machines? We supply the DataNodes the IP of the NameNode container so that it can register with it.

You see errors in log files about connectivity. Well, the container cannot contact the container IP (on other machine), because the outside world cannot directly contact that container.

##### Let's use Host IP

Naturally, you would say let's provide the DataNode container with Host IP of NameNode (the IP of machine on which the NN container is running). The DataNode still binds to its own virtual interface. The NameNode will register it with the container IP (say _10.255.0.1_) and will therefore, cannot contact it either.

So, we want that the containers on different machines are able to contact each other.

### Enter Flannel

What [Flannel](https://github.com/coreos/flannel) does is that it creates an overlay network for containers on 2 or more machines. Each Docker interface is assigned a subnet by flannel. These subnets are coordinated by flannel daemons running on each machine in the network via etcd. A machine, for instance, might get a subnet of _10.255.0.0/24_. Every packet from other machines will be checked, encapsulated and forwarded to this machine by flannel daemon (or by IP Tables). On that machine, another flannel daemon catches this packet, unfolds it and passes it to the specified container. So, the problem of containers not contacting is solved by flannel.

I want to mention that we start Flannel with `--ip-masq=true` and Docker with `--ip-masq=false` options. This way the traffic leaving `docker0` interface will not be masqueraded. However, it will be masqueraded at `flannel` interface. If Docker would masquerade the traffic, , the services on other machines will see it as flannel interface IP due to double masquerading. This will cause problems because, services on different machines cannot communicate with flannel IP. However, the containers would be able to talk to outside world.

There are other solutions, like [Weave](https://github.com/weaveworks/weave), to tackle this problem too. More on this later.

### DNS

All is good up till now. But for due to some uses, the above setup is not complete for Hadoop and HBase (and other) services. Some services outside the container network may contact HDFS or HBase to read data from the DataNodes or RegionServers. Now, if they are bound to their container IP these services outside the network cannot fetch data from the DataNodes.

One solution is to use `/etc/hosts` files inside and outside containers to specify services IP across their host names. But this solution gets clunky. Moreover, it was not possible for us as we were not using `root` user for our Docker containers. Non-root users cannot edit the `hosts` file. The reason for using non-root users was to provide basic security guarantees, related to user interaction with Hadoop clusters.

_Enter DNS_. Other solution, which we adopted was DNS. The services outside the network will have to use DNS to get the IPs of hosts on which a particular DataNode container is running. Note that, this DNS is separate from the DNS for container overlay network (constructed using flannel). We used [SkyDNS](https://github.com/skynetservices/skydns) as it was [etcd](https://github.com/coreos/etcd) backed, which was already being used in our setup.

One problem while setting up DNS was that the containers were not contacting the SkyDNS. We solved this problem by changing the priority of lookup by adding this line to our base Docker images:

```
RUN sed -i s/"files dns"/"dns files"/ /etc/nsswitch.conf
```

It simply swaps the priority from `files` first to `dns` first.

Also, these properties for HBase should be set for DNS awareness:

```
# The IPC address is the address that corresponds to the DNS entry of HBase server container.
hbase.master.ipc.address,
hbase.master.dns.nameserver,
hbase.regionserver.dns.nameserver
```

You can also start Docker with `--dns` option to make all containers DNS aware by default. The IPC address is the address that corresponds to the DNS entry of HBase server container.

##### Updating the DNS

We wrote scripts to update DNS entries. These scripts run at the start of container and update the _etcd_ for SkyDNS.

### What's More

In this post, I have focused on explaining the networking problems for setting up the HBase and similar services. I would like to mention that this all is not done manually. The process is automated by scripts and other software. For container orchestration, we use [Kubernetes](http://kubernetes.io), which provides a very effective way of controlling container specifications and their behavior for large scale clusters.

I also intend to benchmark different approaches for setting up networks Hadoop clusters.



### Did You Like it?

Please, leave comments and any suggestions on improving the method we are using to set up networking in our environment.