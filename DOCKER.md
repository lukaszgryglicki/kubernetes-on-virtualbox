# To use Docker CRI
  - Install docker (which uses containerd), [reference](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).
  - `apt-get update && apt-get install -y apt-transport-https ca-certificates curl software-properties-common gnupg2`.
  - `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -`.
  - `add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"`.
  - `apt-get update && apt-get install -y containerd.io=1.2.13-2`
  - `apt-get install -y docker-ce=5:19.03.11~3-0~ubuntu-$(lsb_release -cs) docker-ce-cli=5:19.03.11~3-0~ubuntu-$(lsb_release -cs)`.
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
  - `systemctl enable docker`.


# Old kubeadm init

- Run on ddmaster: `kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=10.13.13.101`.
