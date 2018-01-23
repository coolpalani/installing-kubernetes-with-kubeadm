# Installing kubernetes with kubeadm
Description of instalation of Kubernetes 1.9 with kubeadm in Centos 7 in an environment with a master node and two minions, behind a corporate proxy. I've created a 3 VM's with Proxmox, using Centos 7 in each one of them. These three are behind a corporate proxy.

## Version of centos
CPE OS Name: cpe:/o:centos:centos:7<br />
Kernel: Linux 3.10.0-229.7.2.el7.x86_64<br />

## IPv4 adressess
centos_master_ip=10.1.114.251<br />
centos_minion1_ip=10.1.114.252<br />
centos_minion2_ip=10.1.114.253<br />
corporate_proxy=http://10.1.14.89:3128/<br />


## Step 1, Set environment variables 
``
#############################################
# SET VARS
#############################################
export centos_master_ip=10.1.114.251
export centos_minion1_ip=10.1.114.252
export centos_minion2_ip=10.1.114.253
export corporate_proxy=http://10.1.14.89:3128/
#############################################
# CONFIGURING ENVIRONMENT
#############################################
cat >>/etc/hosts <<EOL
${centos_master_ip} centos-master 
${centos_minion1_ip} centos-minion-1 
${centos_minion2_ip} centos-minion-2 
EOL
#PROXY CONFIGURATION
cat >/etc/environment <<EOL
ftp_proxy=${corporate_proxy}
http_proxy=${corporate_proxy}
https_proxy=${corporate_proxy}
no_proxy=localhost,127.0.0.0/8,::1,${centos_master_ip},${centos_minion1_ip},${centos_minion2_ip}
EOL
cat /etc/environment
cat >>/etc/profile <<EOL
export ftp_proxy=${corporate_proxy}
export http_proxy=${corporate_proxy}
export https_proxy=${corporate_proxy}
export no_proxy=localhost,127.0.0.0/8,::1,${centos_master_ip},${centos_minion1_ip},${centos_minion2_ip}
EOL
cat /etc/profile
sudo systemctl restart network

#############################################
#TURN OFF ALL SWAP DEVICES 
#############################################
swapoff -a

logout
``
Log in again and execute the following <br />
``
#############################################
#INSTALLING AND CONFIGURING DOCKER
#############################################
yum install -y docker
systemctl enable docker && systemctl start docker

#Configuring docker behind proxy
sudo mkdir -p /etc/systemd/system/docker.service.d
cat >>/etc/systemd/system/docker.service.d/http-proxy.conf <<EOL
[Service]
Environment="HTTP_PROXY=${http_proxy}"
EOL
sudo systemctl daemon-reload
sudo systemctl restart docker

#############################################
#INSTALLING KUBEADM, KUBELET AND KUBECTL
#############################################
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
``


Now, you should check the following: 
*Connectivity of Docker (try docker pull nginx or something else)
*Status of Docker service with ``systemctl status docker``. This sould be active and running.
*Status of kubelet service with ``systemctl status kubelet``. The status of this service may be FAILURE. Check out with ``journalctl -xe`` the cause of failure. The cause should be: "error: unable to load client CA file /etc/kubernetes/pki/ca.crt: open /etc/kubernetes/pki/ca.crt: no such file or directory"
*Also, you can check the versions of docker and kubernetes, as follows:
docker --version
kubelet --version

In my case, the output are the following: <br />
[root@centos-minion-2 ~]# docker --version <br />
Docker version 1.12.6, build ec8512b/1.12.6 <br />
[root@centos-minion-2 ~]# kubelet --version <br />
Kubernetes v1.9.2


At this point, Docker, Kubeadm, Kubelet and Kubectl are installed and properly configure.







