# Ubuntu
## Install Docker and Kubernetes on all servers.
1. The first thing that we are going to do is use SSH to log in to all machines. Once we have logged in, we need to elevate privileges using _sudo_.
```
sudo su
```
2. Verify the MAC address and product_uuid are unique for every node.
```
ip link
cat /sys/class/dmi/id/product_uuid
```
3. Disable swap
   1. Identify configured swap devices and files with ```cat /proc/swaps```.
   2. Turn off all swap devices and files with ```swapoff -a```.
   3. Remove any matching reference found in */etc/fstab*.
   4. **Optional**: Destroy any swap devices or files found in step 1 to prevent their reuse. Due to your concerns about leaking sensitive information, you may wish to consider performing some sort of secure wipe.
4. Ensure that the Docker dependencies are satisfied.
```
apt update && apt install apt-transport-https ca-certificates curl software-properties-common
```
5. Add the Docker repo and install Docker.
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
apt update && apt install docker-ce
```
6. Set the cgroup driver for Docker to systemd, then reload systemd, enable and start Docker
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "data-root": "/opt/docker-data"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

systemctl daemon-reload
systemctl restart docker
```
### *Check with
```
systemctl status docker
docker info | grep -i cgroup
```
7. Install Kubernetes apt repo.
```
apt install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
8. Install Kubernetes.
```
apt update
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```
9. Restart kubelet.
```
systemctl daemon-reload
systemctl restart kubelet
```
### *Note: Complete the following section on the MASTER ONLY!
10. Initialize the cluster using the IP range for Flannel.
```
kubeadm init --pod-network-cidr=10.244.0.0/16
```
11. Copy the *kubeadmn join* command that is in the output. We will need this later.
12. Exit _sudo_ and copy the _admin.conf_ to your home directory and take ownership.
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
13. Deploy Flannel.
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
14. Check the cluster state.
```
kubectl get pods --all-namespaces
```
### *Note: Complete the following steps on the NODES ONLY!
15. Run the _join_ command that you copied earlier (you can change ip to hostname), this requires running the command as sudo on the nodes. Then check your nodes from the master.
```
kubectl get nodes
```
## Create and scale a deployment using kubectl.
1. Create a simple deployment.
```
kubectl create deployment nginx --image=nginx
```
2. Inspect the pod.
```
kubectl get pods
```
3. Scale the deployment.
```
kubectl scale deployment nginx --replicas=4
```
4. Inspect the pods. You should now have 4.
```
kubectl get pods
```
## Deploy dashboard
1. Deploy.
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
```
2. You can access Dashboard using the kubectl command-line tool by running the following command
```
kubectl proxy
```
### *Note: can be run in a separate screen
```
screen
kubectl proxy

#reattach to the screen
screen -r
```
3. Open tunnel from local machine
```
ssh -L 8001:127.0.0.1:8001 <user@server-ip>
```
4. It is accessible in browser with the following url
```
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```
5. Get token
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep kubernetes-dashboard-token | awk '{print $1}') | grep 'token:' | awk -F " " '{print $2}'
```
## Create dashboard account (optional)
1. Create user.
```
cat > dashboard-user.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
EOF

kubectl create -f dashboard-user.yaml
```
2. Make the user a cluster-admin.
```
cat > admin-user-binding.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF

kubectl create -f admin-user-binding.yaml
```
3. Retrieve the token.
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```