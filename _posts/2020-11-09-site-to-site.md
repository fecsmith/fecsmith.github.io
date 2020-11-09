---
title: "One tunnel IPSec Site to site VPN with AWS and Strongswan"
layout: post
date: 2020-11-09 22:05
headerImage: false
tag:
- VPN
- AWS
- strongswan
- IPSec
category: blog
author: kosyanyanwu
description: How to set up one tunnel IPSec Site to site VPN on AWS and a VM on another cloud provider running Strongswan
---

## Introduction
This article outlines the steps to set up a one tunnel IPSec Site to site VPN on AWS and a VM on another cloud provider running Strongswan.
I used a bare metal machine running on [Packet](https://www.packet.com/) when I tried this out.

## Spin up VM and install strongswan
Create an small server with Ubuntu 18.01 LTS, and associate your public SSH key with it to be able to SSH.
```sh
# ssh into server
$ sudo apt update
$ sudo apt install strongswan     # strongswan will be active(running) by default
# Version of strongswan in this case is 5.6.2-1
```
We will refer to this VM henceforth as `Strongswan VPN server`.

## Set up VPN connection on AWS
On AWS, Select the VPC service from the list of Services.

Create a VPC (using a private IP address range e.g 172.17.0.0/16).

On the `Virtual Private Network(VPN)` section on the left, select `Customer Gateways`.

Click `Create Customer Gateway`, on `Routing`, select `Static`, then add the internet-routable IP address of the Strongswan VPN server.

Next select `Virtual Private Gateways` on the section on the left.

Click on `Create Virtual Private Gateway` (VPG), leave the `Amazon default ASN` checked. Then click `Create Virtual Private Gateway`.

Select the VPG you created, click `Actions`, then click `Attach to VPC`.

Attach to the VPC you created earlier.

Following the creation of your VPG, visit the `Route Tables` section under `Virtual Private Cloud`.

Select the routing table corresponding to your subnet(s). (In this case, select the one associated with your VPC)

Afterwards, click the `Route Propagation` tab and then select the `vgw identifier` for the virtual private gateway that was created earlier.

Click `Edit route propagation` to view the `Propagate` checkbox, click the checkbox and choose `Save`.

Update security group to allow access to instances in your VPC from your network. Enable SSH, RDP and ICMP access.

To do that, in the navigation pane, choose `Security Groups` under `Security`.

Select the default security group for the VPC and enable SSH, RDP and ICMP access.

Finally, visit the `VPN` section on the left, choose `Site-to-Site VPN Connections`.

Click `Create VPN Connection`.

In the dialog that results, select the `virtual private gateway` (vgw) and the `customer gateway` that you have previously created.

For routing options, select `Static` then enter the IP CIDR of the Strongswan VPN server (e.g 147.75.44.166/31) i.e enter all of the IP prefixes of your on-premises network.

Leave the `Tunnel Options` to be automatically generated by Amazon.

Then click `Create VPN Connection`.

After the VPN connection has been created, the State of the connection should switch to available in a few minutes.

As the connections have not been made yet to the VPN servers, it is perfectly normal for the icons to be red.

Select the VPN connection that was created, and then note the Tunnel 1 and Tunnel 2 IP addresses.

Click the `Download Configuration` button when finished.

In the dialog box, select `Strongswan` for `Vendor`, Platform `Ubuntu 16.04` (this is all I saw), Software `Strongswan 5.5.1+`.

Then click `Download`.

## Continue setup on Strongswan VPN server
Follow the instructions on the downloaded file from AWS.

I did the steps in IPSEC Tunnel #1, did not do setup for Tunnel #2. (You can do the setup for Tunnel 2 as well for high availability.)

I omitted steps #2 - #5 then moved on to the `Automated Tunnel Healthcheck and Failover` section of the document.

## Checking connectivity
On Strongswan VPN server
```sh
$ root@kosy:/etc# ipsec status       # to ensure both of your tunnels are ESTABLISHED
Security Associations (1 up, 0 connecting):
     Tunnel1[1]: ESTABLISHED 56 seconds ago, 147.75.102.25[147.75.102.25]...35.156.127.214[35.156.127.214]
     Tunnel1{1}:  INSTALLED, TUNNEL, reqid 1, ESP in UDP SPIs: c59e6b8b_i 5ad4d103_o
     Tunnel1{1}:   0.0.0.0/0 === 0.0.0.0/0

$ root@kosy:/etc# ip route       # to ensure route table entries were created for each of your tunnel interfaces, and the destination is the default via 147.75.102.24 dev bond0 onlink
10.0.0.0/8 via 10.80.166.0 dev bond0
10.80.166.0/31 dev bond0 proto kernel scope link src 10.80.166.1
147.75.102.24/31 dev bond0 proto kernel scope link src 147.75.102.25
169.254.40.124/30 dev Tunnel1 proto kernel scope link src 169.254.40.126
172.17.0.0/16 dev Tunnel1 scope link metric 100

$ root@kosy:/etc# iptables -t mangle -L -n       # to ensure entries were made for both of your tunnels in both the INPUT and FORWARD chain
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination

Chain INPUT (policy ACCEPT)
target     prot opt source               destination
MARK       esp  --  35.156.127.214       147.75.102.25        MARK set 0x64

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
TCPMSS     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp flags:0x06/0x02 TCPMSS clamp to PMTU

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination

$ root@kosy:/etc# ifconfig        # to ensure the correct 169.254.x addresses were assigned to each end of your peer-to-peer virtual tunnel interfaces
Tunnel1: flags=209<UP,POINTOPOINT,RUNNING,NOARP>  mtu 1419
        inet 169.254.40.126  netmask 255.255.255.252  destination 169.254.40.125
        inet6 fe80::200:5efe:934b:6619  prefixlen 64  scopeid 0x20<link>
        tunnel   txqueuelen 1000  (IPIP Tunnel)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 6  dropped 0 overruns 0  carrier 6  collisions 0

bond0: flags=5187<UP,BROADCAST,RUNNING,MASTER,MULTICAST>  mtu 1500
        inet 147.75.102.25  netmask 255.255.255.254  broadcast 255.255.255.255
        inet6 2604:1380:2001:2e00::1  prefixlen 127  scopeid 0x0<global>
        inet6 fe80::ec4:7aff:fee5:48fe  prefixlen 64  scopeid 0x20<link>
        ether 0c:c4:7a:e5:48:fe  txqueuelen 1000  (Ethernet)
        RX packets 5110  bytes 3775495 (3.7 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 7377  bytes 765096 (765.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

bond0:0: flags=5187<UP,BROADCAST,RUNNING,MASTER,MULTICAST>  mtu 1500
        inet 10.80.166.1  netmask 255.255.255.254  broadcast 255.255.255.255
        ether 0c:c4:7a:e5:48:fe  txqueuelen 1000  (Ethernet)

enp0s20f0: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 0c:c4:7a:e5:48:fe  txqueuelen 1000  (Ethernet)
        RX packets 5110  bytes 3775495 (3.7 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 7377  bytes 765096 (765.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device memory 0xdf120000-df13ffff

enp0s20f1: flags=6147<UP,BROADCAST,SLAVE,MULTICAST>  mtu 1500
        ether 0c:c4:7a:e5:48:ff  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device memory 0xdf100000-df11ffff

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

On AWS VPN `Site-to-Site VPN Connections` section, check that the status of the one of the tunnels in `Tunnel details` is `UP` (since we made just one tunnel).

## Test traffic on tunnel
### On AWS

Create an AWS t2.nano EC2 instance in your VPC. (Your VPC should already have a subnet else you will be asked to create one)

Let's call this instance `AWS test host`.

Verify that there is an internet gateway attached to your VPC.

Otherwise, choose `Create Internet Gateway` to create an internet gateway.

Select the internet gateway you created, and then choose `Attach to VPC` and follow the directions to attach it to your VPC.

In the navigation pane, choose `Subnets`, and then select your subnet.

On the `Route Table` tab, verify that there is a route with `0.0.0.0/0` as the destination and the internet gateway for your VPC as the target.

Otherwise, do the following:

Choose the ID of the route table (rtb-xxxxxxxx) to navigate to the route table.

On the `Routes` tab, choose `Edit routes`.

Choose `Add route`, use `0.0.0.0/0` as the destination and the internet gateway as the target.

Choose `Save routes`.

You should now be able to ssh into your instance.

If you still cannot ssh into your instance, check the instance inbound rules for SSH.

### On Packet (or your Cloud Provider)

Create a new server. We will call this `Strongswan test host`.

This means we have two machines from Packet. Previous was `Strongswan VPN server` and new one is `Strongswan test host`.

In this scenario, both machines are running Ubuntu 18.04.

We need to configure routing tables on both of them (static routes in this case since we are not using BGP).

SSH into Strongswan VPN server.

Add route to the tunnel IP on Packet side for requests going to the AWS VPC side.

```sh
# ip route add AWS_VPC_CIDR via PACKET_TUNNEL_IP
# e.g
ip route add 172.17.0.0/16 via 169.254.40.126
```

SSH into Strongswan test host.

As at when I tried this, Packet creates machines in separate subnets by default, so trying to fix routing tables here is somehow complicated. Strongswan test host can ping Strongswan VPN server but cannot reach AWS test host. In this case, we need to figure out how to tell the routing table of Strongswan test host that any request to anything in our AWS VPC should be routed through Strongswan VPN server. But how can this be done when both machines are on different subnets?

[Packet](https://www.packet.com/)'s L3-only networking creates some challenges with routing packets from our private strongswan cluster to AWS. In this case, hosts are not in the same subnet. Instead, the host only sees a /31 network with an invisible datacenter-internal routing peer. In order to route packets to and from the VPN, we need to set up a tunnel between [Packet](https://www.packet.com/) nodes and the VPN server. Those tunnels work peer-to-peer; i.e. while we need only one tunnel endpoint per node, the VPN server needs to have tunnel endpoints for all [Packet](https://www.packet.com/) nodes.

You can find an example configuration (server and one node) below. Note that the interface names `node-1` (for Strongswan test host) and `vpn-server` are arbitrary. We will use 192.168.0.0/30 for the internal tunnel transfer network, and we will assume that the VPN server private IP is 10.80.166.1, and the node's IP is 10.80.166.3.
```sh
# VPN Server

vpnserver$ ip tunnel add node-1 mode ipip remote 10.80.166.3 local 10.80.166.1
vpnserver$ ip link set up dev node-1
vpnserver$ ip address add 192.168.0.1 dev node-1
vpnserver$ ip route add 192.168.0.0/30 dev node-1
vpnserver$ iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to 10.80.166.3
# Note that the last line (iptables) creates a source NAT rule which hides the internal IPIP tunnel transfer network from the AWS network.

# Node

node1$ ip tunnel add vpn-server mode ipip remote 10.80.166.1 local 10.80.166.3
node1$ ip link set up dev vpn-server
node1$ ip address add 192.168.0.2 dev vpn-server
node1$ ip route add 192.168.0.0/30 dev vpn-server
```

With this, you should be able to ping the AWS test host from Strongswan test host using their private IP addresses.


### References:

* https://docs.aws.amazon.com/vpn/latest/s2svpn/SetUpVPNConnections.html

* https://openvpn.net/vpn-server-resources/extending-vpn-connectivity-to-amazon-aws-vpc-using-aws-vpc-vpn-gateway-service/