# kubernetes-on-virtualbox

Run Kubernetes (4 node cluster) on a local VirtualBox


# Setup

- Create an Ubuntu 20.04 LTS machine with >= 2G RAM and >= 2 CPU, dynamically sized VDI disk at least 40G.
- Call that installation `master` (VirtualBox name can be `Ubuntu20-mater`), hostname used here: `vmubuntu20-master`.
- Configure 1st network adapter: NAT. Configure port forwarding from guest `22` to host `9922` (SSH access).
- Configure 2nd network adapter: internal network 'intnet'.
- Install Ubuntu20, then docker, kubectl, kubeadm:
  - As user configured on setup, configure root password: `sudo passwd root` - first put your user password then root password twice.
  - Enable root login `vim /etc/ssh/sshd_config` add line `PermitRootLogin yes`, then `sudo service sshd restart`.
  - Login as root.
  - Run: `apt update && apt upgrade`.
  - Kuberentes needs this: `swapoff -a`, `vim /etc/fstab` - remove swap line and swap file.
  - `lsmod | grep br_netfilter`.
  - Run:
  ```
  cat <<EOF > /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  ```
  - `apt-get update && apt-get install -y apt-transport-https ca-certificates curl software-properties-common gnupg2`.
  - `add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu eoan stable"` (no `focal` repo yet, in the future: `add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"`).
  - `apt-get update && apt-get install -y containerd.io=1.2.13-1`.
  - `apt-get install -y docker-ce=5:19.03.8~3-0~ubuntu-eoan && apt-get install -y docker-ce-cli=5:19.03.8~3-0~ubuntu-eoan && docker version`.
  - Run:
  ```
  cat > /etc/docker/daemon.json <<EOF
  {
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "100m"
    },
    "storage-driver": "overlay2"
  }
  EOF
  ```
  - `mkdir -p /etc/systemd/system/docker.service.d; systemctl daemon-reload; systemctl restart docker; service docker status; service containerd status`.
  - `` curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl ``.
  - `chmod +x ./kubectl; mv ./kubectl /usr/local/bin/kubectl; kubectl version --client; kubectl completion bash`.
  - `curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -`.
  - Run:
  ```
  cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
  deb https://apt.kubernetes.io/ kubernetes-xenial main
  EOF
  ```
  - `apt-get update && apt-get install -y kubelet kubeadm`.
  - `sudo apt-mark hold kubelet kubeadm kubectl`.
  - `systemctl daemon-reload; systemctl restart kubelet`.
  - `hostnamectl set-hostname vmubuntu20-master`.
  - `apt install -y nfs-common`.
  - `shutdown -h now`.
- Create VBox DHCP server (run on host): `VBoxManage dhcpserver add --netname intnet --ip 10.13.13.100 --netmask 255.255.255.0 --lowerip 10.13.13.101 --upperip 10.13.13.254 --enable`.
- Clone `master` with linked option (which will use snapshots to save host disk space). Let's call them `node-0`, `node-1`, `node-2`. VM names like `Ubuntu20-node-N`, hostnames `vmubuntu20-node-N`.
- On each of nodes modify port forward from different port 992N --> ssh (to allow SSH access from host). `node-0`: 9923->22, `node-1`: 9924->22, `node-2`: 9925->22.
- Stop all machines from VirtualBox GUI and then start all of them in headless mode (right click, Start -> Headless Start).
- Shell to master and nodes from host via: `ssh -p 992N root@localhost`, replace N=2 for master and then N=3, 4, 5 for nodes.
- On each (N=0, 1, 2):
  - `hostnamectl set-hostname vmubuntu20-node-N`.
  - `apt install net-tools ifupdown`.
  - Edit 2nd interface file (internal network) /etc/network/interfaces.d/enp0s8 (replace N with 1 for master and then 2, 3, 4 - for node(s)):
  ```
  iface enp0s8 inet static
    address 10.13.13.10N
    netmask 255.255.255.0
    gateway 10.13.13.100
    post-up ip route del default via 10.13.13.100 dev enp0s8
  ```
  - Eventually test sequence: `ip addr flush dev enp0s8; ifdown enp0s8; ifup enp0s8`.
  - Edit /etc/hosts on master and node(s), add:
  ```
  10.13.13.101 vmubuntu20-master
  10.13.13.102 vmubuntu20-node-0
  10.13.13.103 vmubuntu20-node-1
  10.13.13.104 vmubuntu20-node-2
  ```
  - Run on master: `kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=10.13.13.101`.
  - Run on master:
  ```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```
  - Save kubeadm join command output to `join.sh` on master and all nodes, something like then `chmod +x join.sh`:
  ```
  #!/bin/bash
  kubeadm join 10.13.13.0:1234 --token xxxxxx.yyyyyyyyyyyy --discovery-token-ca-cert-hash sha256:0123456789abcdef0
  ```
  - On master: `wget https://docs.projectcalico.org/manifests/calico.yaml; kubectl -f calico.yaml`.
  - On master: `kubectl taint nodes --all node-role.kubernetes.io/master-`.
  - On master: `kubectl get po -A; kubectl get nodes`.
  - On all nodes: `./join.sh`.
  - Copy config from master to all nodes:
    - `sftp root@vmubuntu20-node-N`.
    ```
    mkdir .kube
    lcd .kube
    cd .kube
    mput config
    ```
    - `echo 'alias k=kubectl' >> ~/.bashrc`.
    - `k get node; service kubelet status`.

You have 4-node up-to-date Kubernetes cluster running on 4 VirtualBox VMs.

# DevStats

Installing [DevStats](https://github.com/cncf/devstats-helm) on this cluster:

- Label nodes: `for node in vmubuntu20-master vmubuntu20-node-0 vmubuntu20-node-1 vmubuntu20-node-2; do k label node $node node=devstats-app; k label node $node node2=devstats-db; done`.
- Install Helm: `wget https://get.helm.sh/helm-v3.1.2-linux-amd64.tar.gz; tar zxvf helm-v3.1.2-linux-amd64.tar.gz; mv linux-amd64/helm /usr/local/bin; rm -rf linux-amd64/ helm-v3.1.2-linux-amd64.tar.gz`.
- Add Helm charts repository: `helm repo add stable https://kubernetes-charts.storage.googleapis.com/`.
- Install OpenEBS: `k create ns openebs; helm install --namespace openebs openebs stable/openebs --version 1.8.0; helm ls -n openebs`.
- Configure default storage class (need to wait for all OpenEBS pods to be running `k get po -n openebs -w`): `k patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`.
- Install NFS provisioner (for shared NFS volumes access): `helm install local-storage-nfs stable/nfs-server-provisioner --set=persistence.enabled=true,persistence.storageClass=openebs-hostpath,persistence.size=40Gi,storageClass.name=nfs-openebs-localstorage`.
- Create DevStats test and prod namespaces: `k create ns devstats-test; k create ns devstats-prod`.
- Edit `/root/.kube/config` on all nodes and master, make sure you have under contexts:
```
- context:
    cluster: kubernetes
    namespace: devstats-prod
    user: kubernetes-admin
  name: prod
- context:
    cluster: kubernetes
    namespace: devstats-test
    user: kubernetes-admin
  name: test
- context:
    cluster: kubernetes
    namespace: default
    user: kubernetes-admin
  name: shared
```
- Switch to `test` context: `kubectl config use-context test`.
- Install `nginx-ingress`: `helm install --namespace devstats-test nginx-ingress-test stable/nginx-ingress --set controller.ingressClass=nginx-test,controller.scope.namespace=devstats-test,defaultBackend.enabled=false,controller.livenessProbe.initialDelaySeconds=60,controller.livenessProbe.periodSeconds=40,controller.livenessProbe.timeoutSeconds=20,controller.livenessProbe.successThreshold=1,controller.livenessProbe.failureThreshold=5,controller.readinessProbe.initialDelaySeconds=60,controller.readinessProbe.periodSeconds=40,controller.readinessProbe.timeoutSeconds=20,controller.readinessProbe.successThreshold=1,controller.readinessProbe.failureThreshold=5`.
- Switch to `prod` context: `kubectl config use-context prod`.
- Install `nginx-ingress`: `helm install --namespace devstats-prod nginx-ingress-prod stable/nginx-ingress --set controller.ingressClass=nginx-prod,controller.scope.namespace=devstats-prod,defaultBackend.enabled=false,controller.livenessProbe.initialDelaySeconds=60,controller.livenessProbe.periodSeconds=40,controller.livenessProbe.timeoutSeconds=20,controller.livenessProbe.successThreshold=1,controller.livenessProbe.failureThreshold=5,controller.readinessProbe.initialDelaySeconds=60,controller.readinessProbe.periodSeconds=40,controller.readinessProbe.timeoutSeconds=20,controller.readinessProbe.successThreshold=1,controller.readinessProbe.failureThreshold=5`.
- Switch to `shared` context: `kubectl config use-context shared`.
- Create `metallb-system` namespace: `k create ns metallb-system`.
- Create MetalLB configuration - specify `master` IP for `test` namespace and `node-0` IP for `prod` namespace, create file `metallb-config.yaml` and apply if `k apply -f metallb-config.yaml`:
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: metallb-config
data:
  config: |
    address-pools:
    - name: prod
      protocol: layer2
      addresses:
      - 10.13.13.102/32
    - name: test
      protocol: layer2
      addresses:
      - 10.13.13.101/32
```
- Install MetalLB load balancer: `helm install --namespace metallb-system metallb stable/metallb`.
- Check if both test and prod load balancers are OK (they shoudl have External-IP values equal to requested in config map: `kubectl -n devstats-test get svc -o wide -w nginx-ingress-test-controller; kubectl -n devstats-prod get svc -o wide -w nginx-ingress-prod-controller`.
- TODO: skipping `cert-manager` because my iMac doesn't have DNS name configured for its static IP address.
- TODO: to be continued...
