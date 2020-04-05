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
  - Kuberentes needs this: `swapoff -a`, `lsmod | grep br_netfilter`.
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
  - `curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl`.
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
  -  `shutdown -h now`.
- Create VBox DHCP server (run on host): `VBoxManage dhcpserver add --netname intnet --ip 10.13.13.100 --netmask 255.255.255.0 --lowerip 10.13.13.101 --upperip 10.13.13.254 --enable`.
- Clone `master` with linked option (which will use snapshots to save host disk space). Let's call them `node-0`, `node-1`, `node-2`. VM names like `Ubuntu20-node-N`, hostnames `vmubuntu20-node-N`.
- On each of nodes modify port forward from different port 992N --> ssh (to allow SSH access from host). `node-0`: 9923->22, `node-1`: 9924->22, `node-2`: 9925->22.
- Stop all machines from VirtualBox GUI and then start all of them in headless mode (righ click, start -> start in headless mode).
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
```
  - Edit /etc/hosts on master and node(s), add:
```
10.13.13.101 vmubuntu20-master
10.13.13.102 vmubuntu20-node-0
10.13.13.103 vmubuntu20-node-1
10.13.13.104 vmubuntu20-node-2
```
