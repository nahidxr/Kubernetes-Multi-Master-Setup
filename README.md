# Kubernetes Multi-Master Setup with HAProxy and Keepalived

This guide outlines the steps to set up a high-availability Kubernetes multi-master cluster using HAProxy for load balancing and Keepalived for managing a Virtual IP (VIP) between two HAProxy nodes. This setup ensures that the Kubernetes control plane remains highly available, and traffic is distributed across the master nodes.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [IP & Hostname Setup on Ubuntu Machines](#ip--hostname-setup-on-ubuntu-machines)
3. [System Update on All Nodes](#system-update-on-all-nodes)
4. [Install HAProxy & Keepalived on Both Load Balancers](#install-haproxy--keepalived-on-both-load-balancers)
5. [Configure HAProxy](#configure-haproxy)
6. [Start and Enable HAProxy](#start-and-enable-haproxy)
7. [Configure Keepalived](#configure-keepalived)
8. [Start and Enable Keepalived](#start-and-enable-keepalived)
9. [Verify VIP Status](#verify-vip-status)
10. [Kubernetes Master Initialization](#kubernetes-master-initialization)
11. [Conclusion](#conclusion)

## Prerequisites

- **Two HAProxy servers (LB1 and LB2)**
- **Three Kubernetes master nodes**
- **Three Kubernetes worker nodes**
- All nodes should have **Ubuntu 22.04** installed.
- A working **Kubernetes cluster setup** with `kubeadm` and **Calico** as the network provider.
- Ensure all nodes can resolve each other via internal hostnames.

## IP & Hostname Setup on Ubuntu Machines

#### On HAProxy Node 1 (LB1):
```bash
$ hostnamectl set-hostname LB1
