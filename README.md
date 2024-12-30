# Kubernetes Cluster Setup with MariaDB Galera and NFS on AWS
This document outlines the steps to set up a Kubernetes cluster with 1 master node and 2 worker nodes using **kubeadm**, deploy **MariaDB** with **Galera** for database clustering, and configure **NFS** as persistent storage in AWS.
## Prerequisites
- AWS account with sufficient permissions.
- Ubuntu 20.04/22.04 AMI on EC2 instances.
- Kubernetes `kubeadm`, `kubelet`, `kubectl`, and Docker installed on each instance

## Step 1: Provision EC2 Instances
### Create EC2 Instances:
1. **Master Node**:
   - Instance Type: t2.medium (or higher).
   - Security Group: Open ports `6443`, `2379-2380`, `10250-10252`, and `10255`.
2. **Worker Nodes** (2):
   - Instance Type: t2.medium (or higher).
   - Security Group: Same as Master Node.
3. **NFS Server**:
   - Instance Type: t2.medium.
   - Security Group: Open ports `2049` and `111` for NFS.
## Step 2: Prepare All Nodes
1. SSH into each node and update the system:
   ``` bash
   sudo apt update && sudo apt upgrade -y
2. Install dependencies
   ``` bash
   sudo apt install apt-transport-https curl -y
   sudo apt install -y docker.io
   sudo systemctl enable docker
   sudo systemctl start docker
3. Install Kubernetes components
   ``` bash
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   sudo apt update
   sudo apt install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
4. Disable swap (Kubernetes requirement):
   ``` bash
   sudo swapoff -a
   sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
5. Load necessary kernel modules:
   ``` bash
   sudo modprobe overlay
   sudo modprobe br_netfilter
6. Enable br_netfilter and configure iptables
   ``` bash
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   br_netfilter
   EOF
   sudo modprobe br_netfilter
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF
   sudo sysctl --system
## Step 3: Initialize Kubernetes Cluster
1. Initialize the cluster (run only on master node):
   ``` bash
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16
   ```
   After this command you get configure kubectl and Join worker nodes to the cluster commands.
2. Configure kubectl:
   ``` bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   export KUBECONFIG=/path/to/cluster-config
3. Install Flannel network plugin (run only on master node) or Install Calico network plugin
   - Flannel
   ``` bash
   kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
   ```
   - Calico
   ``` bash
   kubectl apply -f https://docs.projectcalico.org/v3.25/manifests/calico.yaml

## Step 4: Join Worker Nodes to the Cluster
1. Use the kubeadm join command provided by the kubeadm init output on the master node.
    ``` bash
    kubeadm join 172.31.19.36:6443 --token 922x9d.v0jn4c8he0s286js --discovery-token-ca-cert-hash sha256:8897fd8eb97f2ea0686ccf7507f287ffffd5cf681496fb324940330561c80e4c
2. Verify the installation:
    ``` bash
    kubectl get nodes
    kubectl get pods --all-namespaces
## Step 5: Set Up NFS Server
1. SSH into the NFS Server and install NFS server:
   ``` bash
   sudo apt install -y nfs-kernel-server
2. Create an NFS export directory:
   ``` bash
   sudo mkdir -p /srv/nfs/kubedata
   sudo chmod -R 777 /srv/nfs/kubedata
3. Configure the NFS export:
   ``` bash
   echo "/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
   sudo exportfs -a
4. Start NFS server:
   ``` bash
   sudo systemctl enable nfs-server
   sudo systemctl start nfs-server
## Step 6: Configure NFS in Kubernetes
1. SSH into all Kubernetes nodes and install nfs-common:
   ``` bash
   sudo apt install -y nfs-common
2. Create a PersistentVolume (PV) and PersistentVolumeClaim (PVC) for NFS storage:
``` bash
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  nfs:
    path: /srv/nfs/kubedata
    server: <NFS_SERVER_IP>
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```
3. Apply the YAML configuration:
 ``` bash
kubectl apply -f nfs-pv-pvc.yaml
```
## Step 7: Deploy MariaDB with Galera
1. Create a StatefulSet for MariaDB Galera:
``` bash
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb-galera
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mariadb
  serviceName: mariadb-galera
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.5
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpassword"
        volumeMounts:
        - name: mariadb-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mariadb-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```
2. Apply the StatefulSet configuration:
``` bash
kubectl apply -f mariadb-galera.yaml
```
## Step 8: Verify Deployment
1. Check the status of the Pods:
   ``` bash
   kubectl get pods -o wide
2. Verify MariaDB is running:
   ``` bash
   kubectl exec -it <mariadb-pod> -- mysql -u root -p
3. Verify the NFS-mounted storage in a Pod:
   ``` bash
   kubectl exec -it <mariadb-pod> -- mysql -u root -p
   ```
## Step 9: Conclusion
- You have successfully set up a Kubernetes cluster with a Master and Worker Nodes.
- MariaDB with Galera is deployed for database clustering.
- NFS is configured as persistent storage for Kubernetes.
## Usefull Commands.
``` bash
 kubectl cluster info
```
``` bash
kubectl get nodes
```
``` bash
kubectl get pods --all-namespaces
```
``` bash
kubectl get pv
```
``` bash
kubectl get pvc
```
``` bash
kubectl get pods
```
``` bash
kubectl describe pod <pod name>
```
``` bash
kubectl exec -it <mariadb-pod-name> -- mysql -u root -p
```
``` bash
kubectl exec -it <mariadb-pod-name> -- df -h
```
``` bash
kubectl exec -it <mariadb-pod-name> -- ls -l /var/lib/mysql
```
``` bash
kubectl logs <pod-name>
```
``` bash
kubectl get pods -n kube-system | grep calico
```
``` bash
kubectl exec -it <pod-name> -- ping <other-pod-IP>
```
``` bash
kubectl scale deployment <deployment-name> --replicas=5
```
``` bash
kubectl autoscale deployment <deployment-name> --cpu-percent=70 --min=2 --max=10
```



 
   
    
