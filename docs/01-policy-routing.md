Policy Routing
===

## Table of Contents
- [Overview](#overview)
- [Requirements](#requirements)
- [Layout](#layout)
  - [Diagram](#diagram)
  - [Network Namespace](#network-namespace)
- [Hardware](#hardware-and-software)
- [Implementation](#implementation)
  - [Step 01 - Create the virtual machine](#step-01---create-the-virtual-machine)
  - [Step 02 - Access the vm](#step-02---access-the-vm)
  - [Step 03 - Create the required namespaces](#step-03---create-the-required-namespaces)

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
graph TB
    Internet@{ label: "Internet<br>ns-internet<br>8.8.8.8", shape: cloud }
    ISP_1["ns-isp-1<br>10.0.1.1/30"]
    ISP_2["ns-isp-2<br>10.0.2.1/30"]
    subgraph "ns-router"
        subgraph "router"
            ROUTER_PORT_1["router-port-1<br>10.0.1.2/30"]
            ROUTER_PORT_2["router-port-2<br>192.168.100.1/24"]
            ROUTER_PORT_3["router-port-3<br>10.0.2.2/30"]
        end
        SWITCH["br-lan<br>unmanaged switch<br>no IP address"]        
    end

    PC_1["ns-pc-1<br>192.168.100.101/24"]
    PC_2["ns-pc-2<br>192.168.100.102/24"]
    PC_3["ns-pc-3<br>192.168.100.103/24"]
    
    Internet <--> ISP_1
    Internet <--> ISP_2
    ISP_1 <--> ROUTER_PORT_1
    ISP_2 <--> ROUTER_PORT_3
    ROUTER_PORT_2 <--> SWITCH
    SWITCH <--> PC_1
    SWITCH <--> PC_2
    SWITCH <--> PC_3
```
### Network Namespace
- ns-internet - This will represent a common destination being accessed by different ISPs.
- ns-isp-1 and ns-isp-2 - The namespace `ns-isp-1` will be the default gateway for `ns-pc-1`, `ns-pc-2` and `ns-pc-3`. We will then introduce policy routing where `ns-pc-1` will be explicitly configured to use `ns-isp-1`, `ns-pc-2` will be explicitly configured to use `ns-isp-2` and `ns-pc-3` will use `ns-isp-1` implicitly as PC-3 was not explcitly configure with any policy routing and will use the default route.
- ns-router - The router has 3 ports:
  - 10.0.1.2/30 - This connects to `ns-isp-1`
  - 10.0.2.2/30 - This connects to `ns-isp-2`.
  - 192.168.101.1/24 - This is the gateway for `ns-pc-1`, `ns-pc-2` and `ns-pc-3`.
  - The `link` of type `bridge` will act as a non-managed switch, without any IP addresses where `ns-pc-1`, `ns-pc-2` and `ns-pc-3` connects to this `bridge`.
- ns-pc-1, ns-pc-2 and ns-pc-3 - These will represent individual devices that connect to the switch, `br-lan`.

## Hardware and Software
- This was tested on a `Beelink Mini S` with `Intel(R) N100` with `16 GB DDR4`.
- Also tested with `multipass` version `1.16.1` on M2 Mac Book Pro with 32 GB RAM.

## Implementation
### Step 01 - Create the virtual machine
Command:
```
echo  'package_update: true\npackage_upgrade: true\npackages:\n  - traceroute' | multipass launch --name lab1 --cloud-init -
```
Sample output:
```
$ echo  'package_update: true\npackage_upgrade: true\npackages:\n  - traceroute' | multipass launch --name lab1 --cloud-init -
Launched: lab1
```

### Step 02 - Access the vm
Command:
```
multipass shell lab1
```
Sample output:
```
$ multipass shell lab1
Welcome to Ubuntu 24.04.3 LTS (GNU/Linux 6.8.0-90-generic aarch64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Tue Dec 23 18:35:49 MST 2025

  System load:             0.0
  Usage of /:              52.3% of 3.80GB
  Memory usage:            20%
  Swap usage:              0%
  Processes:               94
  Users logged in:         0
  IPv4 address for enp0s1: 192.168.2.8
  IPv6 address for enp0s1: fde8:bf1d:8203:1850:5054:ff:fe92:d4cc


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@lab1:~$ 
```
### Step 03 - Create the required namespaces
```
ip netns list
sudo ip netns add ns-internet
sudo ip netns add ns-isp-1
sudo ip netns add ns-isp-2
sudo ip netns add ns-router
sudo ip netns add ns-pc-1
sudo ip netns add ns-pc-2
sudo ip netns add ns-pc-3
ip netns list
```
Sample output:
```
ubuntu@lab1:~$ ip netns list
ubuntu@lab1:~$ sudo ip netns add ns-internet
ubuntu@lab1:~$ sudo ip netns add ns-isp-1
ubuntu@lab1:~$ sudo ip netns add ns-isp-2
ubuntu@lab1:~$ sudo ip netns add ns-router
ubuntu@lab1:~$ sudo ip netns add ns-pc-1
ubuntu@lab1:~$ sudo ip netns add ns-pc-2
ubuntu@lab1:~$ sudo ip netns add ns-pc-3
ubuntu@lab1:~$ ip netns list
ns-pc-3
ns-pc-2
ns-pc-1
ns-router
ns-isp-2
ns-isp-1
ns-internet
ubuntu@lab1:~$ 
```


