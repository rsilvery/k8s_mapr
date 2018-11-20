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

### Install MapR Volume Driver
Installing the MapR Volume Driver allows you to create persistent volumes that map to volumes in your MapR filespace.
* Download the most recent version of the following files to your directory on your K8s master node from [here](http://package.mapr.com/tools/KubernetesDataFabric/):
  * kdf-namespace.yaml
  * kdf-rbac.yaml
  * kdf-plugin-centos.yaml
  * kdf-provisioner.yaml
* Change host IP (labeled: *changeme!*) in kdf-plugin-centos.yaml to your Master host IP (can get with *hostname --ip-address*)
* Create the following resources as shown:
  ```
  kubectl create -f kdf-namespace.yaml
  kubectl create -f kdf-rbac.yaml
  kubectl create -f kdf-plugin-centos.yaml
  kubectl create -f kdf-provisioner.yaml
  ```

### Configure namespace, secret, and data access
These are the initial steps needed to configure data cluster access for KubeFlow
* Create a Namespace for Kubeflow: 
  ```
  kubectl create ns kubeflow
  ```
* Create Secret for cluster access (see [kf-secret.yaml](kf-secret.yaml)):
  * Get long lived service ticket from a MapR cluster. Can follow steps [here](https://mapr.com/docs/61/SecurityGuide/GeneratingServiceTicket.html)
  * Base64 encode this ticket. You can use a webtool like [this](https://www.base64encode.org/)
  * Insert encoded ticket string into [kf-secret.yaml](kf-secret.yaml) 
  * Create secret: 
  ```
  kubectl create -f kf-secret.yaml
  ```
* Create Persistent Volume (PV) to provision storage in the cluster
  * Edit [kf-pv.yaml](kf-pv.yaml) and enter your cluster info where indicated under "options"
  ```
  kubectl create -f kf-pv.yaml
  ```
* Create Persistent Volume Claim (PVC) to bind to this claim (using [kf-pvc.yaml](kf-pvc.yaml))
  ```
  kubectl create -f kf-pvc.yaml
  ```


If you want to test that this worked, you can use the [kf-testpod.yaml](kf-testpod.yaml) to generate a Centos pod with this mount.