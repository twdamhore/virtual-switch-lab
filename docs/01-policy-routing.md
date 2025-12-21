Policy Routing
===

## Table of Contents
- [Overview](#overview)
- [Requirements](#requirements)
- [Layout](#layout)
  - [Diagram](#diagram)
  - [Network Namespace](#network-namespace)
- [Hardware](#hardware)
- [Implementation](#implementation)
  - [Step 01 - Create the virtual machine](#step-01---create-the-virtual-machine)
  - [Step 02 - Create the required namespaces](#step-02---create-the-required-namespaces)
  - 

## Overview
The lab uses multipass to create a virtual machine.
There will be 6 network namespaces that will be from various PC at home and a layer as mock internet.
The objective is to contrast not using policy routing vs using policy routing.
The PC at home will ping internet using the default route.
Then policy routing will be added.
The PC at home will ping internet again to show that policy routing has been implemented.


## Requirements
- [Multipass](https://canonical.com/multipass) installation.
- The vm will be created with default configuration of 1 core and 1 GB of RAM and 4 GB of disk space.

## Layout
### Diagram
```mermaid
graph TD
    Internet("Internet")
    
    ISP1["ISP-1<br>ns-isp-1<br>- 10.0.1.1/30"]
    ISP2["ISP-2<br>ns-isp-2<br>- 10.0.2.1/30"]
    
    Bridge["Router<br>ns-router<br>- 192.168.1.1/24<br>- 10.0.1.2/30<br>- 10.0.2.2/30"]
    
    subgraph LAN
        PC1["PC-1<br>ns-pc-1<br>- 192.168.1.101/24"]
        PC2["PC-2<br>ns-pc-2<br>- 192.168.1.102/24"]
        PC3["PC-3<br>ns-pc-3<br>- 192.168.1.103/24"]
    end
    
    Internet <---> ISP1
    Internet <---> ISP2
    ISP1 <---> Bridge
    ISP2 <---> Bridge
    Bridge <---> PC1
    Bridge <---> PC2
    Bridge <---> PC3
```
### Network Namespace
- ns-net - This is mock Internet tha will be the final destination for PC-1, PC-2 and PC-3.
- ns-isp-1 and ns-isp-2 - The namespace `ns-isp-1` will be the default gateway for PC-1, PC-2 and PC-3. We will then introduce policy routing where PC-1 will be explicitly configured to use `ns-isp-1`, PC-2 will be explicitly configured to use `ns-isp-2` and PC-3 will use `ns-isp-1` implicitly as PC-3 was not explcitly configure with any policy routing and will use the default route.
- ns-router - The router connects 3 networks together (LAN, ISP-1, ISP-2). The default route and policy routing will be configured on this interface.
- ns-pc-1, ns-pc-2 and ns-pc-3 - This will represent devices connected to `ns-router`.

## Hardware
This was tested on a `Beelink Mini S` with `Intel(R) N100` with `16 GB DDR4`.

## Implementation
### Step 01 - Create the virtual machine
Command:
```bash
multipass launch --name lab1
```
This will create a new virtual machine with `1 GB` RAM and 1 thread.

Run the `shell` command to access the virtual machine.
```bash
multipass shell lab1
```
### Step 02 - Create the required namespaces
Verify that there are no existing network namespaces.

```code
ubuntu@lab1:~$ ip link show type netns
```
This list will be empty.

Create 3 namespace for the 3 PC devices.
```
ubuntu@lab1:~$ sudo ip netns add ns-pc-1
ubuntu@lab1:~$ sudo ip netns add ns-pc-2
ubuntu@lab1:~$ sudo ip netns add ns-pc-3
```

Create 1 namespace for the router.
```
ubuntu@lab1:~$ sudo ip netns add ns-router
```

Verify all namespaces are present
```
ubuntu@lab1:~$ ip netns show
ns-isp-2
ns-isp-1
ns-router
ns-pc-3
ns-pc-2
ns-pc-1
```
### Step 03 - Assign an IP address to ns-pc-1
Check current IP address for `ns-pc-1`:
```
ubuntu@lab1:~$ sudo ip netns exec ns-pc-1 ip -4 -br address show
```
Nothing.

---
Check current links for `ns-pc-1`:
```
ubuntu@lab1:~$ sudo ip netns exec ns-pc-1 ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
The link `lo` is down.

---
Add a new link of type `veth`. This is like a network cable with 2 endpoints. One goes `ns-pc-1` and one goes to `ns-router`.
```
ubuntu@lab1:~$ ip link show type veth
ubuntu@lab1:~$ sudo ip link add veth-pc-1-pc type veth peer name veth-pc-1-rt
ubuntu@lab1:~$ ip link show type veth
3: veth-pc-1-rt@veth-pc-1-pc: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 5e:c2:a2:af:31:19 brd ff:ff:ff:ff:ff:ff
4: veth-pc-1-pc@veth-pc-1-rt: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 56:c6:06:80:c2:2f brd ff:ff:ff:ff:ff:ff
```
The these are peers of each other, `veth-pc-1-rt` is the peer of `eth-pc-1-pc`.

---
Plug in the cables.
```
ubuntu@lab1:~$ ip link show type veth
3: veth-pc-1-rt@veth-pc-1-pc: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 5e:c2:a2:af:31:19 brd ff:ff:ff:ff:ff:ff
4: veth-pc-1-pc@veth-pc-1-rt: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 56:c6:06:80:c2:2f brd ff:ff:ff:ff:ff:ff
ubuntu@lab1:~$ sudo ip netns exec ns-router ip link show type veth
ubuntu@lab1:~$ sudo ip netns exec ns-router ip link show type veth
ubuntu@lab1:~$ sudo ip link set veth-pc-1-pc netns ns-pc-1
ubuntu@lab1:~$ sudo ip link set veth-pc-1-rt netns ns-router
ubuntu@lab1:~$ sudo ip netns exec ns-pc-1 ip link show type veth
4: veth-pc-1-pc@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 56:c6:06:80:c2:2f brd ff:ff:ff:ff:ff:ff link-netns ns-router
ubuntu@lab1:~$ sudo ip netns exec ns-router ip link show type veth
3: veth-pc-1-rt@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 5e:c2:a2:af:31:19 brd ff:ff:ff:ff:ff:ff link-netns ns-pc-1
ubuntu@lab1:~$ ip link show type veth
```
The link `veth-pc-1-pc` is in now in `ns-pc-1` and the link `veth-pc-1-rt` is in `ns-router`.
