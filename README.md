I did this on a single AWS t2.2xlarge instance with the following initial config:
* Kubenetes v1.12.2

### Configure node
* Create repo /etc/yum.repos.d/kubernetes.repo
    ```
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    ```
* Disable SE Linux: 
    ```
    sudo setenforce 0
    sudo sed -i '/^SELINUX./ { s/enforcing/disabled/; }' /etc/selinux/config
    ```
* Disable Swap
    ```
    sudo swapoff -a
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    ```
* Enable Bridge Networking:
  * Create file */etc/sysctl.d/k8s.conf* with the following content
    ```
    net.bridge.bridge-nf-call-ip6tables = 1
	net.bridge.bridge-nf-call-iptables = 1
    ```
  * Apply change:
    ```
    sudo sysctl --system
    ```

### Install Docker
```
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker

```

### Install Kubernetes
* Install core components:
```
sudo yum install -y kubelet kubeadm kubectl
sudo systemctl start kubelet
sudo systemctl enable kubelet

```
* Initialize Kubernetes master:
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=$(hostname --ip-address)

```
* Deploy KubeConfig:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
* Install Pod Network (I chose Flannel):
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
* If you're running this on a single node, untaint master node:
```
kubectl taint node --all node-role.kubernetes.io/master:NoSchedule-
```
* Deploy Web UI:
```
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```
* Create service for WebUI for external access:
```
kubectl -n kube-system edit service kubernetes-dashboard (# Change type from 'ClusterIP' to 'NodePort')
```
* Create default user:
```
kubectl create clusterrolebinding --user system:serviceaccount:kube-system:default kube-system-cluster-admin --clusterrole cluster-admin
```
* Get login token:
```
kubectl describe serviceaccount default -n kube-system
kubectl describe secret default-token -n kube-system
```
* Get port:
```
kubectl -n kube-system get service kubernetes-dashboard
```
* Visit Web UI on this port 


