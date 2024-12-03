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

## Configurations
```bash
On HAProxy Node 1 (LB1):
$ hostnamectl set-hostname LB1

On HAProxy Node 2 (LB2):
$ hostnamectl set-hostname LB2

On Kubernetes Master Node 1 (kube-master1):
$ hostnamectl set-hostname kube-master1

On Kubernetes Master Node 2 (kube-master2):
$ hostnamectl set-hostname kube-master2

On Kubernetes Master Node 3 (kube-master3):
$ hostnamectl set-hostname kube-master3

Update /etc/hosts on all nodes:
$ sudo vim /etc/hosts

Add the following entries for all nodes in the cluster:
192.168.5.220 kube-master1
192.168.5.221 kube-master2
192.168.5.227 kube-master3
192.168.5.224 LB1
192.168.5.225 LB2

System Update on All Nodes
Update and upgrade the system packages:

$ sudo apt update && sudo apt upgrade -y

Install HAProxy & Keepalived on Both Load Balancers

Install HAProxy:
$ sudo apt-get install haproxy -y

Install Keepalived:
$ sudo apt-get install keepalived -y

Configure HAProxy
Edit the HAProxy Configuration File (/etc/haproxy/haproxy.cfg):
$ sudo vim /etc/haproxy/haproxy.cfg


Add the following configuration to load balance traffic to the Kubernetes master nodes:
global
    log /dev/log local0 warning
    chroot /var/lib/haproxy
    pidfile /var/run/haproxy.pid
    maxconn 4000
    user haproxy
    group haproxy
    daemon
    stats socket /var/lib/haproxy/stats

defaults
    log global
    option httplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend kubernetes
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server kubemaster1 192.168.5.220:6443 check fall 3 rise 2
    server kubemaster2 192.168.5.221:6443 check fall 3 rise 2
    server kubemaster3 192.168.5.227:6443 check fall 3 rise 2

Start and Enable HAProxy

Start HAProxy:
$ sudo systemctl start haproxy

Enable HAProxy to start on boot:
$ sudo systemctl enable haproxy

Restart HAProxy (if necessary):
$ sudo systemctl restart haproxy

Configure Keepalived
Edit Keepalived Configuration File (/etc/keepalived/keepalived.conf):


$ sudo vim /etc/keepalived/keepalived.conf
Add the following configuration for virtual IP (VIP) management:


# Define the script used to check if haproxy is still working
vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

# Configuration for the virtual interface
vrrp_instance VI_1 {
    interface ens19  # Replace with the actual network interface name
    state BACKUP     # Set this to BACKUP on the second load balancer
    priority 100     # Set this to 100 on the second load balancer
    virtual_router_id 51
    smtp_alert       # Activate email notifications (optional)
    authentication {
        auth_type AH
        auth_pass myPassw0rd  # Set this to a secret password
    }

    # The virtual IP address shared between the two load balancers
    virtual_ipaddress {
        192.168.5.224  # This is the virtual IP (VIP)
    }

    # Use the script above to check if HAProxy should failover
    track_script {
        chk_haproxy
    }
}

Note: Interface will be based on IP address interface. Priority will be different in both proxy server(ex:100,120). Virtual ip is free ip address which is not use in anywhere.

Start and Enable Keepalived
Start Keepalived:
$ sudo systemctl start keepalived

Enable Keepalived to start on boot:
$ sudo systemctl enable keepalived

Restart Keepalived (if necessary):
$ sudo systemctl restart keepalived

Verify VIP Status
Verify the virtual IP (VIP) is up:


$ ip a show ens19  # Replace ens19 with the actual network interface
If the VIP is down or not assigned correctly, check the status of Keepalived and HAProxy:


$ sudo systemctl status keepalived
$ sudo systemctl status haproxy

Kubernetes Master Initialization
On the first master node, run the following kubeadm command to initialize the Kubernetes control plane:


$ kubeadm init --control-plane-endpoint="192.168.5.224:6443" --upload-certs --apiserver-advertise-address=192.168.5.20 --pod-network-cidr=192.168.0.0/16

