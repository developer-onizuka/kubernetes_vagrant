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
$ vagrant init generic/ubuntu2010
```

# 3. Edit Vagrantfile
You might use the file of Vagrantfile located in the /mnt/vagrant/ubuntu below:
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2010"
  config.vm.provider :libvirt do |kvm|
    kvm.memory = 4096
    kvm.cpus = 2
  end
#-------------------- master --------------------#
  config.vm.define "master" do |server|
    server.vm.network "private_network", ip: "192.168.33.100"
    server.vm.hostname = "master"
    server.vm.provision "shell", inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "swap.img.swap"
      sudo sed -ie "12d" /etc/fstab
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      cat <<EOF > /etc/docker/daemon.json
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
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
      sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.33.100
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      kubectl taint nodes --all node-role.kubernetes.io/master-
      kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
      curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
      chmod 700 get_helm.sh
      ./get_helm.sh
      helm repo add stable https://charts.helm.sh/stable
    SHELL
  end
#-------------------- worker1 --------------------#
  config.vm.define "worker1" do |server|
    server.vm.network "private_network", ip: "192.168.33.101"
    server.vm.hostname = "worker1"
    server.vm.provision "shell", inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "swap.img.swap"
      sudo sed -ie "12d" /etc/fstab
      apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      cat <<EOF > /etc/docker/daemon.json
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
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
    SHELL
  end
#-------------------- worker2 --------------------#
  config.vm.define "worker2" do |server|
    server.vm.network "private_network", ip: "192.168.33.102"
    server.vm.hostname = "worker2"
    server.vm.provision "shell", inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "swap.img.swap"
      sudo sed -ie "12d" /etc/fstab
      apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      cat <<EOF > /etc/docker/daemon.json
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
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
    SHELL
  end
#-------------------- haproxy --------------------#
  config.vm.define "haproxy" do |server|
    server.vm.network "private_network", ip: "192.168.33.10"
    server.vm.network "private_network", ip: "192.168.133.10"
    server.vm.hostname = "haproxy"
    server.vm.provision "shell", inline: <<-SHELL
      apt-get update
      sudo apt-get install -y docker.io
      sudo apt-get install -y sshpass
      ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa <<< y
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.100
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.101
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.102
    SHELL
  end
#-------------------------------------------------#
end
```

# 4. Vagrant Up for all VMs
```
$ vagrant up --provider=libvirt
```

# 5. Login to Haproxy VM via PuTTy
The login name and password are "vagrant" as default.
```
$ ssh vagrant@192.168.33.10
vagrant@192.168.33.10's password: 
Last login: Sun Sep 19 13:08:55 2021 from 192.168.33.1
vagrant@haproxy:~$ 
```

# 6. SSH settings from HAproxy
```
$ ssh-keygen -t rsa
$ ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.100
$ ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.101
$ ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.102
```

# 7. Labeling on Master node
```
sudo kubectl label node worker1 node-role.kubernetes.io/node=worker1
sudo kubectl label node worker2 node-role.kubernetes.io/node=worker2
```


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
