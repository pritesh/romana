# Step to install Kubernetes 1.5.2 with Romana

## Installing Kubernetes

```bash
# On Controller
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y docker.io
sudo apt-get install -y kubeadm
sudo kubeadm init --use-kubernetes-version v1.5.2
# now you would get something like this at the end:
# kubeadm join --token=<token> <ip-address>

# On Nodes
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y docker.io
sudo apt-get install -y kubeadm
# Use the kubeadm join command and token from controller here.
sudo kubeadm join --token=<token> <ip-address>
```

## Installing kubectl 1.5
```bash
# kubectl still defaults to the default set by kubeadm
# thus we need to install the latest version manually.
wget https://storage.googleapis.com/kubernetes-release/release/v1.5.2/bin/linux/amd64/kubectl
sudo chown --reference=/usr/bin/kubectl kubectl
sudo chmod --reference=/usr/bin/kubectl kubectl
sudo mv kubectl /usr/bin/kubectl
# now use kubectl version to see if client and server are both
# of same version.
kubectl version
```

## Installing Romana

```bash
kubectl apply -f https://raw.githubusercontent.com/pritesh/romana/tw/kubeadm-install/romana.yml
```

## Changing Kubernetes ClusterIP Range

Kurbernetes will change the clusterIP Range if manifest for kube-apiserver
is changed, so following line does the trick, where 10.112.0.0/12 is the
new clusterIP Range.
```bash
sudo sed -i.bak 's/--service-cluster-ip-range=*.*\"/--service-cluster-ip-range=10.112.0.0\/12\"/' /etc/kubernetes/manifests/kube-apiserver.json
```

## Scheduling Pods on Kubernetes master

Removing taint from kubernetes master allows scheduling pods on master.
This is useful when single node is present.

```bash
kubectl taint nodes --all dedicated:NoSchedule-
```

## Changing/Adding labels to nodes by patching it.

```bash
kubectl patch node <node name> -p '{"metadata":{"labels":{"kubeadm.alpha.kubernetes.io/role": "master"}}}'
````

## Testing if the install is working
```bash
# First try checking the pods and see if all of them
# come up properly and are in running state. You may
# want to wait few minutes before it settles, there
# may be some restarts but eventually it does come up.
kubectl get pods -a -o wide --all-namespaces

# once all pods are up, try installing cirros and see
# if it comes up and if dns works correctly.
kubectl run cirros --image=cirros --replicas=4

# check if the cirros pods are running
kubectl get pods -a -o wide 

# once they are running login into them using
kubectl exec -it <pod-name> /bin/sh

# once inside the pod, use nslookup to test dns
nslookup kubernetes.default
# the result would be as follows:
#Server:    10.96.0.10
#Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
#
#Name:      kubernetes.default
#Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local

nslookup cirros-4036794762-9s46o
#Server:    10.96.0.10
#Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
#
#Name:      cirros-4036794762-9s46o
#Address 1: 100.112.11.131 cirros-4036794762-9s46o

```

## Bringup and Expose service to external world
```bash
# bringup nginx with 2 replicas on port 80.
kubectl run nginx --image=nginx --replicas=2 --port=80

# Check if the pods are up else wait few seconds for them
# to start, when desired and current columns both show
# correct number of replicas,  it means nginx is up and
# running correctly.
kubectl get deployments --all-namespaces

# Now expose the service to outside world.
kubectl expose deployment nginx --port=80 --type=LoadBalancer --external-ip=192.168.99.10

# Check if the configuration was applied correctly, if
# it was, then external IP should be reflected using
# following command.
kubectl get services --all-namespaces -o wide

# Test if nginx is up and running
curl 192.168.99.10:80
#<!DOCTYPE html>
#<html>
#<head>
#<title>Welcome to nginx!</title>
#<snip>...

# You are all set and ready to roll.
```

### Removing cirros and nginx deployments
```bash
kubectl delete deployments nginx cirros
```

### Removing kubernetes install.
```bash
# On Controller and all nodes, run following command
# to reset your cluster to what it was before installing
# kubernetes
# BEWARE, YOU WILL LOSE ALL YOUR PODS/SERVICES/DATA/etc.
sudo kubeadm reset
```
### Deleting All docker Containers and Images

```bash
sudo docker rm $(sudo docker ps -a -q)
sudo docker rmi $(sudo docker images -a -q)
````

### Find the kubeadm token for new nodes to join the clsuter

```bash
kubectl -n kube-system get secret clusterinfo -o yaml | grep token-map | awk '{print $2}' | base64 -d | sed "s|{||g;s|}||g;s|:|.|g;s/\"//g;" | xargs echo
````

### Limitations

* Currently it pulls images from dockers and depends on kubeadm
for setup which needs external repos for installation.
* If changes to manifests in `/etc/kubernetes/manifests/` are made
before CNI plugin is installed then kube-api-server fails to come
up.
