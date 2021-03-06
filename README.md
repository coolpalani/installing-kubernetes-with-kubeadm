# Installing Kubernetes 1.9.2 with kubeadm on Centos 7
Description of instalation of Kubernetes 1.9.2 using kubeadm on Centos 7 (1708). The environment is composed of a master node and two minions, behind a corporate proxy. Flannel is used as network plugin.

## IPv4 addresses
centos_master_ip=10.1.114.251<br />
centos_minion1_ip=10.1.114.252<br />
centos_minion2_ip=10.1.114.253<br />
corporate_proxy=http://10.1.14.89:3128/<br />


## Step 1: Set environment variables 
Execute the following in **all the hosts** (Change IPv4 addresses for yours):

    #############################################
    # VARS
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
    no_proxy=localhost,127.0.0.1,127.0.0.0/8,::1,${centos_master_ip},${centos_minion1_ip},${centos_minion2_ip}
    EOL
    
    cat >>/etc/profile <<EOL
    export ftp_proxy=${corporate_proxy}
    export http_proxy=${corporate_proxy}
    export https_proxy=${corporate_proxy}
    export no_proxy=localhost,127.0.0.1,127.0.0.0/8,::1,${centos_master_ip},${centos_minion1_ip},${centos_minion2_ip}
    EOL
    
    sudo systemctl restart network
    
    #############################################
    #TURN OFF ALL SWAP DEVICES 
    #############################################
    swapoff -a
    
    #############################################
    #DISABLE FIREWALLD
    #############################################
    systemctl stop firewalld
    systemctl disable firewalld
   
    #############################################
    #DISABLE SELINUX
    #############################################
    cat >/etc/sysconfig/selinux<<EOL
    # This file controls the state of SELinux on the system.
    # SELINUX= can take one of these three values:
    #       enforcing - SELinux security policy is enforced.
    #       permissive - SELinux prints warnings instead of enforcing.
    #       disabled - SELinux is fully disabled.
    SELINUX=disabled
    # SELINUXTYPE= type of policy in use. Possible values are:
    #       targeted - Only targeted network daemons are protected.
    #       strict - Full SELinux protection.
    SELINUXTYPE=targeted
    EOL
    
**IMPORTANT:** To disable completely the swap, edit the file /etc/fstab, and comment **ONLY** the line corresponding to the swap. Example:<br />

    #
    # /etc/fstab
    # Created by anaconda on Tue Jan 16 09:39:42 2018
    #
    # Accessible filesystems, by reference, are maintained under '/dev/disk'
    # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
    #
    /dev/mapper/centos-root /                        xfs     defaults        0 0
    UUID=ad15df-f2f5-475... /boot                    xfs     defaults        0 0
    #/dev/mapper/centos-swap swap                    swap    defaults        0 0



## Step 2: Installing Docker, Kubeadm, Kubelet and Kubectl
Reboot and Log in again.Then execute the following in **all the hosts**:<br />

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

## Step 3: Check the configuration and installation of the previous components

Now, you should check the following: 
* Check status of **Firewall** (``systemctl stop firewalld;systemctl disable firewalld``)
* Check status of **SElinux** (with ``sestatus``) The output should be disabled.
* Status of **Docker service** with ``systemctl status docker``. This sould be active and running.
* Connectivity of Docker (try docker pull nginx or something else).
* Status of **Kubelet service** with ``systemctl status kubelet``. The status of this service **MAY BE FAILURE**. Check out with ``journalctl -xe`` the cause of failure. The cause should be: **"error: unable to load client CA file /etc/kubernetes/pki/ca.crt: open /etc/kubernetes/pki/ca.crt: no such file or directory"**
* Also, you can check the versions of docker and kubernetes, as follows:
docker --version
kubelet --version

In my case, the output are the following: <br />
*[root@centos-minion-2 ~]# docker --version*<br />
Docker version 1.12.6, build ec8512b/1.12.6 <br />
*[root@centos-minion-2 ~]# kubelet --version*<br />
Kubernetes v1.9.2

At this point, Docker, Kubeadm, Kubelet and Kubectl are installed and properly configure.<br />

## Step 4: Initialize the master
#### Start Kubernetes on master
Execute on the **master**, the following:<br />
``kubeadm init –apiserver-advertise-address=10.1.114.251 –pod-network-cidr=10.244.0.0/16`` <br />
To make kubectl work for your **non-root user**, you might want to run these commands: <br />
    ``mkdir -p $HOME/.kube`` <br />
    ``sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`` <br />
    ``sudo chown $(id -u):$(id -g) $HOME/.kube/config`` <br />
Alternatively, if you are the **root user**, you could run this: <br /> 
    
    export KUBECONFIG=/etc/kubernetes/admin.conf
    cat >>/etc/environment <<EOL
    KUBECONFIG=/etc/kubernetes/admin.conf
    EOL   
    cat >>/etc/profile <<EOL
    export KUBECONFIG=/etc/kubernetes/admin.conf
    EOL

#### Install Flannel
``kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
`` <br />
The output of kubeadm init will be used later to join minions to the cluster. It looks like the following: <br />
**kubeadm join --token *some_token* 10.1.114.251:6443 --discovery-token-ca-cert-hash sha256:*some_sha***

## Step 5: Join the minions
Execute the output of previous step **on the minions** <br />
The ouput looks like the following: <br />

    This node has joined the cluster:
    * Certificate signing request was sent to master and a response
     was received.
    * The Kubelet was informed of the new secure connection details.

## Step 6: Check the installation
There are some steps that I recommend to test the success of the installation.<br />
First, execute the following command in the master:<br />
    ``kubectl get nodes``<br />
In my case, I've encountered with all nodes are in status "Not Ready"<br /> 

    NAME              STATUS     ROLES     AGE       VERSION
    centos-master     NotReady   master    55m       v1.9.2
    centos-minion-1   NotReady   <none>    45m       v1.9.2
    centos-minion-2   NotReady   <none>    45m       v1.9.2

To fix this I did the following:<br /> 

Execute the following on master:<br /> 
    ``kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}' --all-namespaces``<br /> 
If the output is empty, edit /etc/kubernetes/manifests/kube-controller-manager.yaml on the **master node**<br /> 
Add the following lines on the spec of kube-controller-manager command (line 24):<br /> 

    - --allocate-node-cidrs=true
    - --cluster-cidr=10.244.0.0/16
    
Then reload kubelet<br /> 
    ``systemctl daemon-reload;systemctl restart kubelet`` <br /> 

After this fix, the output was the following:<br /> 

    NAME              STATUS     ROLES     AGE       VERSION
    centos-master     Ready      master    55m       v1.9.2
    centos-minion-1   NotReady   <none>    45m       v1.9.2
    centos-minion-2   NotReady   <none>    45m       v1.9.2

Executing <br />

    kubectl describe nodes
The following error was observed in the two minions nodes: <br /> 

    KubeletNotReady runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: ni config uninitialized

To fix this, run the following node in the master:<br /> 

    kubectl patch node centos-master -p '{"spec":{"podCIDR":"10.244.0.0/16"}}'
    kubectl patch node centos-minion-1 -p '{"spec":{"podCIDR":"10.244.0.0/16"}}'
    kubectl patch node centos-minion-2 -p '{"spec":{"podCIDR":"10.244.0.0/16"}}'

and then restart kubelet on **each node**<br /> 
    ``systemctl daemon-reload;systemctl restart kubelet ``<br /> 

After this points, all the nodes are in Ready status

### Check corporate proxy configuration
Now, it is necessary to correctly configure the corporate proxy, in order to access the services and the pods.
There are two possible situations:
1. That you have access to the corporate proxy and can edit its rules.<br /> 
	In this case, you need to define the "no upstream" rules for the pods(10.244.0.0/16) and services (10.96.0.0/12) networks.
2. That the management of the corporate proxy is not within your reach.<br /> 
	In this case, you can install and configure tinyproxy on any of the nodes. In my case I have installed it in the master node, and I have configured it with the "no upstream" rules of the previous case.


