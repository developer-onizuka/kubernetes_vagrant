# kubernetes_vagrant

# 1. Install Vagrant on your Ubuntu
```
$ sudo apt install --yes vagrant vagrant-libvirt
```

# 2. Make dir and Initization vagrant
You can select the OS images which is called as "box" in https://app.vagrantup.com/boxes/search.
```
$ mkdir -p /mnt/vagrant/ubuntu
$ cd /mnt/vagrant/ubuntu
$ vagrant init bento/ubuntu-20.04
```

# 3. Edit Vagrantfile
You might edit the file of Vagrantfile located in the D:\Vagrant\ubuntu20 .
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-20.04"

  config.vm.define "master" do |server|
    server.vm.network "private_network", ip: "192.168.30.100"
    server.vm.hostname = "master"
    server.vm.provision "shell", inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "dev-mapper-vgvagrant\x2dswap_1.swap"
      sudo sed -ie "11d" /etc/fstab
      apt-get update
      apt-get install -y curl
      curl https://get.docker.com | sh && sudo systemctl --now enable docker
      cat <<EOF | sudo tee /etc/docker/daemon.json
      {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
          "max-size": "100m"
        },
        "storage-driver": "overlay2"
      }
      EOF
      sudo systemctl enable docker
      sudo systemctl daemon-reload
      sudo systemctl restart docker
    SHELL
  end
  config.vm.define "worker1" do |server|
    server.vm.network "private_network", ip: "192.168.30.101"
    server.vm.hostname = "worker1"
  end
  config.vm.define "worker2" do |server|
    server.vm.network "private_network", ip: "192.168.30.102"
    server.vm.hostname = "worker2"
  end
  config.vm.define "haproxy" do |server|
    server.vm.network "private_network", ip: "192.168.30.10"
    server.vm.network "private_network", ip: "192.168.199.10"
    server.vm.hostname = "haproxy"
  end
  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
  
    # Customize the amount of memory on the VM:
    vb.memory = "4096"
    vb.cpus = "2"
  end
end
```

# 4. Up all VMs
```
$ vagrant up
```

# 5. Login to Haproxy VM via PuTTy
The login name and password are "vagrant" as default.
```
login as: vagrant
vagrant@192.168.30.10's password:
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun 19 Sep 2021 02:28:54 AM UTC

  System load:  0.0               Users logged in:       0
  Usage of /:   2.3% of 61.31GB   IPv4 address for eth0: 10.0.2.15
  Memory usage: 4%                IPv4 address for eth1: 192.168.30.10
  Swap usage:   0%                IPv4 address for eth2: 192.168.199.10
  Processes:    117


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
vagrant@vagrant:~$
```

# 6. SSH settings from HAproxy
```
cd
sudo hostnamectl set-hostname haproxy
ssh-keygen -t rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.30.100
ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.30.101
ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.30.102
ssh vagrant@192.168.30.100
ssh vagrant@192.168.30.101
ssh vagrant@192.168.30.102
```

# 7. Install Docker on every node
```
sudo apt-get update
sudo apt-get install -y curl
curl https://get.docker.com | sh && sudo systemctl --now enable docker
```




<Master Step#1>
sudo hostnamectl set-hostname master
sudo swapoff -a
sudo systemctl mask "dev-mapper-vgvagrant\x2dswap_1.swap"
sudo systemctl --type swap -all
sudo vi /etc/fstab
sudo reboot
sudo apt-get update
sudo apt-get install -y curl
curl https://get.docker.com | sh && sudo systemctl --now enable docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update \
&& sudo apt-get install -y -q kubelet kubectl kubeadm \
&& sudo kubeadm init --pod-network-cidr=192.168.0.0/16 \
--apiserver-advertise-address=192.168.30.100

mkdir -p $HOME/.kube \
&& sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config \
&& sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl get nodes


<Master Step#2>
kubectl label node worker1 node-role.kubernetes.io/node=worker1
kubectl label node worker2 node-role.kubernetes.io/node=worker2

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml


curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm repo add stable https://charts.helm.sh/stable
helm search repo stable




<Master Step#3>
$ helm install --generate-name hello-world
NAME: hello-world-1631973493
LAST DEPLOYED: Sat Sep 18 13:58:13 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services hello-world-1631973493)
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT





kubectl get services
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hello-world-1631973493   NodePort    10.102.98.109   <none>        80:31627/TCP   33s
kubernetes               ClusterIP   10.96.0.1       <none>        443/TCP        142m

helm delete $(helm ls -n default | awk '/hello-world/{print $1}') -n default





<Worker>
sudo hostnamectl set-hostname worker1
sudo swapoff -a
sudo systemctl mask "dev-mapper-vgvagrant\x2dswap_1.swap"
sudo systemctl --type swap -all
sudo vi /etc/fstab
sudo reboot
sudo apt-get update
sudo apt-get install -y curl
curl https://get.docker.com | sh && sudo systemctl --now enable docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update \
&& sudo apt-get install -y -q kubelet kubectl kubeadm

sudo kubeadm join 192.168.30.100:6443 --token lpjjem.j4j25xepq5ojq2xp \
        --discovery-token-ca-cert-hash sha256:a682fdeb124b82477b2392c41b201b2a0883248e2851677a5c27371596e3f522




cat <<EOF > haproxy.cfg
global
    maxconn 256

defaults
    mode http
    timeout client     120000ms
    timeout server     120000ms
    timeout connect      6000ms

listen http-in
    bind *:80
    server proxy-server 192.168.199.10
EOF

sudo docker run -itd --rm --name haproxy -p 80:80 -v $(pwd)/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro haproxy:1.8

From PC browser http://192.168.30.10
  ```
