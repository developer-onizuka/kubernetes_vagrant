# kubernetes_vagrant
https://youtu.be/64p6ywzwiCM

# 0. Goal

|  | CPU | Memory | GPU | GPU Driver |
| --- | --- | --- | --- | --- |
| Master | 2 | 4,096 MB | no | N/A |
| Worker1 | 2 | 4,096 MB | no | N/A |
| Worker2 | 2 | 4,096 MB | no | N/A |
| haproxy | 2 | 4,096 MB | no | N/A |


# 1. Install Vagrant on your Ubuntu
```
$ sudo apt install --yes vagrant vagrant-libvirt
```

# 2. Make dir and Initization vagrant
You can select the OS images which is called as "box" in https://app.vagrantup.com/boxes/search.
```
$ mkdir -p /mnt/vagrant/k8s
$ cd /mnt/vagrant/k8s
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
  config.vm.define "master_192.168.33.100" do |server|
    server.vm.network "private_network", ip: "192.168.33.100"
    server.vm.hostname = "master"
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "swap.img.swap"
      sudo sed -ie "12d" /etc/fstab
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      sudo apt-get install -y sshpass
      ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa <<< y
      cat <<EOF > ~/.ssh/config
host 192.168.33.*
   StrictHostKeyChecking no
EOF
      chmod 600 ~/.ssh/config
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.101
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.102
      sudo cat <<EOF > /etc/docker/daemon.json
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
      sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
      sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.33.100
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      sudo kubectl taint nodes --all node-role.kubernetes.io/master-
      sudo kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
      token=$(sudo kubeadm token list |tail -n 1 |awk '{print $1}')
      hashkey=$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //')
      ssh vagrant@192.168.33.101 sudo kubeadm join 192.168.33.100:6443 --token $token --discovery-token-ca-cert-hash sha256:$hashkey
      ssh vagrant@192.168.33.102 sudo kubeadm join 192.168.33.100:6443 --token $token --discovery-token-ca-cert-hash sha256:$hashkey
      sudo kubectl label node worker1 node-role.kubernetes.io/node=worker1
      sudo kubectl label node worker2 node-role.kubernetes.io/node=worker2
      #curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
      #chmod 700 get_helm.sh
      #./get_helm.sh
      #helm repo add stable https://charts.helm.sh/stable
    SHELL
  end
#-------------------- worker1 --------------------#
  config.vm.define "worker1_192.168.33.101" do |server|
    server.vm.network "private_network", ip: "192.168.33.101"
    server.vm.hostname = "worker1"
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "swap.img.swap"
      sudo sed -ie "12d" /etc/fstab
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      sudo cat <<EOF > /etc/docker/daemon.json
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
      sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
    SHELL
  end
#-------------------- worker2 --------------------#
  config.vm.define "worker2_192.168.33.102" do |server|
    server.vm.network "private_network", ip: "192.168.33.102"
    server.vm.hostname = "worker2"
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sudo swapoff -a
      sudo systemctl mask "swap.img.swap"
      sudo sed -ie "12d" /etc/fstab
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      sudo cat <<EOF > /etc/docker/daemon.json
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
      sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
    SHELL
  end
#-------------------- haproxy --------------------#
  config.vm.define "haproxy_192.168.133.10" do |server|
    server.vm.network "private_network", ip: "192.168.33.10"
    server.vm.network "private_network", ip: "192.168.133.10"
    server.vm.hostname = "haproxy"
    server.vm.provision "shell", privileged: false, inline: <<-SHELL
      sudo apt-get update
      sudo apt-get install -y docker.io
      sudo apt-get install -y sshpass
      ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa <<< y
      cat <<EOF > ~/.ssh/config
host 192.168.33.*
   StrictHostKeyChecking no
EOF
      chmod 600 ~/.ssh/config
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
You can login to the master-node (192.168.33.100), so you can check if master and workers are ready.
```
vagrant@master:~$ sudo kubectl get nodes
NAME      STATUS   ROLES                  AGE    VERSION
master    Ready    control-plane,master   2m8s   v1.22.2
worker1   Ready    node                   96s    v1.22.2
worker2   Ready    node                   94s    v1.22.2
```

# 5. Helm instal hello-world Using NodePort
```
vagrant@master:~$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
vagrant@master:~$ chmod 700 get_helm.sh
vagrant@master:~$ ./get_helm.sh
vagrant@master:~$ helm repo add stable https://charts.helm.sh/stable
vagrant@master:~$ helm create hello-world
Creating hello-world

vagrant@master:~$ vi hello-world/values.yaml
.....
service:
  type: NodePort      # changed from ClusterIP
  port: 8080          # changed from 80
.....

vagrant@master:~$ cat <<EOF > hello-world/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "hello-world.fullname" . }}
  labels:
    {{- include "hello-world.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
      nodePort: 30001
  selector:
    {{- include "hello-world.selectorLabels" . | nindent 4 }}
EOF

vagrant@master:~$ sudo helm install --generate-name hello-world
NAME: hello-world-1632122557
LAST DEPLOYED: Mon Sep 20 07:22:37 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services hello-world-1632122557)
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT

vagrant@master:~$ sudo kubectl get services
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
hello-world-1632122557   NodePort    10.110.139.187   <none>        8080:30001/TCP   2s
kubernetes               ClusterIP   10.96.0.1        <none>        443/TCP          115m
```
# Access from Host Machine
Blowse http://192.168.33.101:30001/  --> Accessible

Blowse http://192.168.33.102:30001/  --> Accessible

Blowse http://192.168.33.100:30001/  --> Accessible

# (Optional) Using HAproxy
Confirm you can not access http://192.168.133.10 before this step. 

After this step, you can access http://192.168.133.10 from PC. It means that the IP address exposed outside is only 192.168.133.10 and you don't need exposing the 192.168.33.10x:30001 to privent from attacks of internet outside.

```
vagrant@haproxy:~$ curl http://192.168.33.100:30001
vagrant@haproxy:~$ curl http://192.168.33.101:30001
vagrant@haproxy:~$ curl http://192.168.33.102:30001
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

vagrant@haproxy:~$ cat <<EOF > haproxy.cfg
global
    maxconn 256

defaults
    mode http
    timeout client     120000ms
    timeout server     120000ms
    timeout connect      6000ms

listen http-in
    bind *:80
    server proxy-server1 192.168.33.100:30001
    server proxy-server2 192.168.33.101:30001
    server proxy-server3 192.168.33.102:30001
EOF

$ sudo docker run -itd --rm --name haproxy -p 80:80 -v $(pwd)/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro haproxy:1.8
```

# 6. Uninstall hello-world
```
sudo helm delete $(sudo helm ls -n default | awk '/hello-world/{print $1}') -n default
```



