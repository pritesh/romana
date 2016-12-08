# Step to install Kubernetes 1.5 Beta with Romana

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
sudo kubeadm init --use-kubernetes-version v1.5.0-beta.2
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

## Installing Romana

```bash
kubectl apply -f https://raw.githubusercontent.com/pritesh/romana/tw/containerize/kubeadm-install/romana.yml
```

### Limitations

Currently it pulls images from dockers and depends on kubeadm for setup which needs external repos for installation.
