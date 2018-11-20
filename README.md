I did this on a single AWS t2.2xlarge instance with the following initial config:
* Kubenetes v1.12.2

### Configure node
* Create repo /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
* Disable SE Linux: 
```sudo setenforce 0
sudo sed -i '/^SELINUX./ { s/enforcing/disabled/; }' /etc/selinux/config```
* Disable Swap
```sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab```