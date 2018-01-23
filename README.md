# Installing-kubernetes-with-kubeadm
Description of instalation of Kubernetes 1.9 with kubeadm in Centos 7 in an environment with a master node and two minions, behind a corporate proxy. I've created a 3 VM's with Proxmox, using Centos 7 in each one of them. These three are behind a corporate proxy.

# Version of centos
CPE OS Name: cpe:/o:centos:centos:7<br />
Kernel: Linux 3.10.0-229.7.2.el7.x86_64<br />

# IPv4 adressess
centos_master_ip=10.1.114.251<br />
centos_minion1_ip=10.1.114.252<br />
centos_minion2_ip=10.1.114.253<br />
corporate_proxy=http://10.1.14.89:3128/<br />
