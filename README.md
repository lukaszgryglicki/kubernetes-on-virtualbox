# kubernetes-on-virtualbox

Run Kubernetes (4 node cluster) on a local VirtualBox. Currently tested with Kubernetes v1.20.0.


# Setup

- Create an Ubuntu 20.04 LTS machine with >= 2G RAM and >= 2 CPU, dynamically sized VDI disk at least 40G.
- Call that installation `master` (VirtualBox name can be `Ubuntu20-master`), hostname used here: `vmubuntu20-master`.
- Configure 1st network adapter: NAT. Configure port forwarding from guest `22` to host `9922` (SSH access).
- Configure 2nd network adapter: internal network 'intnet'.
- Install Ubuntu 20.04 LTS, then containerd, kubectl, kubeadm:
  - Reference [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).
  - As user configured on setup, configure root password: `sudo passwd root` - first put your user password then root password twice.
  - Enable root login `vim /etc/ssh/sshd_config` add line `PermitRootLogin yes`, then `sudo service sshd restart`.
  - Optional: ACPI shutown VirtualBox UI, once done, start it in headless mode: `VBoxHeadless --startvm Ubuntu20-master`, wait a bit and then `ssh -p9922 root@localhost`.
  - Or: Login as root.
  - Run: `apt update && apt upgrade`.
  - Kuberentes needs this: `swapoff -a`, `vim /etc/fstab` - remove swap line and swap file.
  - `lsmod | grep br_netfilter`. If not loaded then `modprobe br_netfilter`.
  - Run:
```
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
  - `sysctl --system`.
  - Install containerd (this is now a recommended Kubernetes CRI instead of docker).
  - Run:
```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```
  - `sudo modprobe overlay; sudo modprobe br_netfilter`.
  - Run:
```
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
  - `sudo sysctl --system`.
  - `sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common`.
  - `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key --keyring /etc/apt/trusted.gpg.d/docker.gpg add -`.
  - `sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"`.
  - `sudo apt-get update && sudo apt-get install -y containerd.io`.
  - `sudo mkdir -p /etc/containerd`.
  - `sudo containerd config default | sudo tee /etc/containerd/config.toml`.
  - `sudo systemctl restart containerd`.
  - `sudo systemctl enable containerd`.
  - Set cgroup driver to systemd.
  - `vim /etc/containerd/config.toml`, search: `/plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options`.
  - Add: `SystemdCgroup = true`, so it looks like:
```
          base_runtime_spec = ""
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
    [plugins."io.containerd.grpc.v1.cri".cni]
```
  - `service containerd restart`.
  - Install kubectl [reference](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
  - `curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"`.
  - `chmod +x ./kubectl; mv ./kubectl /usr/local/bin/kubectl; kubectl version --client; kubectl completion bash`.
  - `vim ~/.bashrc`, uncomment `. /etc/bash_completion` part, relogin, `echo 'source <(kubectl completion bash)' >>~/.bashrc`, `kubectl completion bash >/etc/bash_completion.d/kubectl`.
  - `echo 'alias k=kubectl' >>~/.bashrc; echo 'complete -F __start_kubectl k' >>~/.bashrc`.
  - Install kubeadm [reference](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl) (we will use calico network plugin, no additional setup is needed).
  - `curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -`.
  - Run:
```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
  - `apt-get update && apt-get install -y kubelet kubeadm kubectl`.
  - `apt-mark hold kubelet kubeadm kubectl`
  - `systemctl daemon-reload; systemctl restart kubelet`.
  - Configure the cgroup driver for kubeadm using containerd:
```
cat <<EOF | sudo tee /etc/kubeadm_cgroup_driver.yml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.13.13.101
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
networking:
  podSubnet: '192.168.0.0/16'
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```
  - Other configuration:
  - `hostnamectl set-hostname vmubuntu20-master`.
  - `apt install -y nfs-common net-tools`.
  - Edit `/etc/hosts` add:
```
10.13.13.101 vmubuntu20-master
10.13.13.102 vmubuntu20-node-0
10.13.13.103 vmubuntu20-node-1
10.13.13.104 vmubuntu20-node-2
```
  - `shutdown -h now`.
- Create VBox DHCP server (run on host): `VBoxManage dhcpserver add --netname intnet --ip 10.13.13.100 --netmask 255.255.255.0 --lowerip 10.13.13.101 --upperip 10.13.13.254 --enable`.
- Or modify existing: `VBoxManage dhcpserver modify --netname intnet --ip 10.13.13.100 --netmask 255.255.255.0 --lowerip 10.13.13.101 --upperip 10.13.13.254 --enable`.
- Clone `master` with linked option (this is on clone step 2, it will use snapshots to save host disk space). Let's call them `node-0`, `node-1`, `node-2`. VM names like `Ubuntu20-node-N`, hostnames `vmubuntu20-node-N`.
- On each of nodes modify port forward from different port 992N --> ssh (to allow SSH access from host). `node-0`: 9923->22, `node-1`: 9924->22, `node-2`: 9925->22.
- Stop all machines from VirtualBox GUI and then start all of them in headless mode (right click, Start -> Headless Start).
- Shell to master and nodes from host via: `ssh -p 992N root@localhost`, replace N=2 for master and then N=3, 4, 5 for nodes.
- On each (N=0, 1, 2):
  - `hostnamectl set-hostname vmubuntu20-node-N`.
  - Configure internal network (N=1,2,3,4): `ifconfig enp0s8 10.13.13.10N netmask 255.255.255.0; route del default enp0s8; route add default gw 10.13.13.100 enp0s8; ifconfig enp0s8 up; ip route del default via 10.13.13.100 dev enp0s8`.
  - save this as `net.sh`.
  - Then: `ifconfig enp0s8 | grep inet; ip r | grep enp0s8`.
  - `ping google.com; ping vmubuntu20-master; ping vmubuntu20-node-0; ping vmubuntu20-node-1; ping vmubuntu20-node-2`.
  - Initialize cluster [reference](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node):
  - Run on master: `kubeadm init --config /etc/kubeadm_cgroup_driver.yml`.
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
  - Install networking plugin (calico):
  - On master: `wget https://docs.projectcalico.org/manifests/calico.yaml; kubectl apply -f calico.yaml`.
  - Allow scheduling on the master node [reference](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation):
  - On master: `kubectl taint nodes --all node-role.kubernetes.io/master-`.
  - On master: `kubectl get po -A; kubectl get nodes`. wait for all pods to be in `Running` state.
  - On all nodes: `./join.sh`.
  - Copy config from master to all nodes:
  - `sftp root@vmubuntu20-node-N`:
```
mkdir .kube
lcd .kube
cd .kube
mput config
```
  - `k get node; service kubelet status`.

You have 4-node up-to-date Kubernetes cluster running on 4 VirtualBox VMs.

# Restarting

If you restart VMs you need to reapply ifconfigs via `./net.sh` and possibly `service kubelet restart` (but it doesn't seem to be needed).

# DevStats

Installing [DevStats](https://github.com/cncf/devstats-helm) on this cluster:

- Label nodes: `for node in vmubuntu20-master vmubuntu20-node-0 vmubuntu20-node-1 vmubuntu20-node-2; do k label node $node node=devstats-app; k label node $node node2=devstats-db; done`.
- Install Helm (master & nodes): `wget https://get.helm.sh/helm-v3.4.2-linux-amd64.tar.gz; tar zxvf helm-v3.4.2-linux-amd64.tar.gz; mv linux-amd64/helm /usr/local/bin; rm -rf linux-amd64/ helm-v3.4.2-linux-amd64.tar.gz`.
- Add Helm charts repository (master & nodes): `helm repo add stable https://charts.helm.sh/stable`.
- Add OpenEBS charts repository (master & nodes): `helm repo add openebs https://openebs.github.io/charts`
- Add Ingress NGINX charts repository (master & nodes): `helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx`.
- Apply Helm repos config (master & nodes): `helm repo update`.
- Install OpenEBS: `k create ns openebs; helm install --namespace openebs openebs openebs/openebs; helm ls -n openebs; kubectl get pods -n openebs`.
- Configure default storage class (need to wait for all OpenEBS pods to be running `k get po -n openebs -w`): `k patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`.
- Install NFS provisioner (for shared NFS volumes access): `helm install local-storage-nfs stable/nfs-server-provisioner --set=persistence.enabled=true,persistence.storageClass=openebs-hostpath,persistence.size=40Gi,storageClass.name=nfs-openebs-localstorage`.
- Create DevStats test and prod namespaces: `k create ns devstats-test; k create ns devstats-prod`.
- Edit `vim /root/.kube/config` on all nodes and master, make sure you have this under `contexts`:
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
- Switch to `test` context: `k config use-context test`.
- Install `nginx-ingress`: `helm install --namespace devstats-test nginx-ingress-test ingress-nginx/ingress-nginx --set controller.ingressClass=nginx-test,controller.scope.namespace=devstats-test,defaultBackend.enabled=false,controller.livenessProbe.initialDelaySeconds=60,controller.livenessProbe.periodSeconds=40,controller.livenessProbe.timeoutSeconds=20,controller.livenessProbe.successThreshold=1,controller.livenessProbe.failureThreshold=5,controller.readinessProbe.initialDelaySeconds=60,controller.readinessProbe.periodSeconds=40,controller.readinessProbe.timeoutSeconds=20,controller.readinessProbe.successThreshold=1,controller.readinessProbe.failureThreshold=5`.
- Optional: edit nginx service: `k edit svc -n devstats-test nginx-ingress-test-ingress-nginx-controller` add annotation: `metallb.universe.tf/address-pool: test` and (very optional) spec: `loadBalancerIP: 10.13.13.101`.
- Switch to `prod` context: `k config use-context prod`.
- Install `nginx-ingress`: `helm install --namespace devstats-prod nginx-ingress-prod ingress-nginx/ingress-nginx --set controller.ingressClass=nginx-prod,controller.scope.namespace=devstats-prod,defaultBackend.enabled=false,controller.livenessProbe.initialDelaySeconds=60,controller.livenessProbe.periodSeconds=40,controller.livenessProbe.timeoutSeconds=20,controller.livenessProbe.successThreshold=1,controller.livenessProbe.failureThreshold=5,controller.readinessProbe.initialDelaySeconds=60,controller.readinessProbe.periodSeconds=40,controller.readinessProbe.timeoutSeconds=20,controller.readinessProbe.successThreshold=1,controller.readinessProbe.failureThreshold=5`.
- Optional: edit nginx service: `k edit svc -n devstats-prod nginx-ingress-prod-ingress-nginx-controller` add annotation: `metallb.universe.tf/address-pool: prod` and (very optional) spec `loadBalancerIP: 10.13.13.102`.
- Switch to `shared` context: `k config use-context shared`.
- Install MetalLB [reference](https://metallb.universe.tf/installation/#installation-by-manifest):
- `kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml`.
- `kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml`.
- `kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"`.
- Create MetalLB configuration - specify `master` IP for `test` and `node-0` IP for `prod`, create file `metallb-config.yaml` and apply if `k apply -f metallb-config.yaml`:
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
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
- More details [here](https://raw.githubusercontent.com/google/metallb/v0.9.5/manifests/example-config.yaml).
- Check if both test and prod load balancers are OK (they should have External-IP values equal to requested in config map: `k -n devstats-test get svc -o wide -w nginx-ingress-test-ingress-nginx-controller; k -n devstats-prod get svc -o wide -w nginx-ingress-prod-ingress-nginx-controller`).
- Note: Helm chart for metalLB is currently broken, see details in `MLB_HELM.md`.
- TODO: skipping `cert-manager` because my iMac doesn't have DNS name configured for its static IP address.
- Switch to `test` context: `k config use-context test`.
- Clone `devstats-helm` repo: `git clone https://github.com/cncf/devstats-helm`, `cd devstats-helm`.
- For each file in `devstats-helm/secrets/*.secret.example` create corresponding `secrets/*.secret` file. Vim saves with end line added, truncate such files via `truncate -s -1 filename`.
- Deploy DevStats secrets: `helm install devstats-test-secrets ./devstats-helm --set skipPVs=1,skipBackupsPV=1,skipVacuum=1,skipBackups=1,skipBootstrap=1,skipProvisions=1,skipCrons=1,skipAffiliations=1,skipGrafanas=1,skipServices=1,skipPostgres=1,skipIngress=1,skipStatic=1,skipAPI=1,skipNamespaces=1`.
- Will be deploying two small projects: KEDA and SMI.
- Deploy storage: `helm install devstats-test-pvcs ./devstats-helm --set skipSecrets=1,skipVacuum=1,skipBackups=1,skipBackupsPV=1,skipBootstrap=1,skipProvisions=1,skipCrons=1,skipAffiliations=1,skipGrafanas=1,skipServices=1,skipPostgres=1,skipIngress=1,skipStatic=1,skipAPI=1,skipNamespaces=1,indexPVsFrom=70,indexPVsTo=72,backupsPVSize=2Gi`.
- Create backups PV(s): `helm install devstats-test-backups-pv ./devstats-helm --set skipSecrets=1,skipPVs=1,skipBackups=1,skipBootstrap=1,skipProvisions=1,skipCrons=1,skipAffiliations=1,skipGrafanas=1,skipServices=1,skipPostgres=1,skipIngress=1,skipStatic=1,skipAPI=1,skipNamespaces=1,backupsPVSize=2Gi`
- Deploy "minimalistic" patroni tweaked for that tiny env: `helm install devstats-test-patroni ./devstats-helm --set skipSecrets=1,skipPVs=1,skipBackupsPV=1,skipVacuum=1,skipBackups=1,skipBootstrap=1,skipProvisions=1,skipCrons=1,skipAffiliations=1,skipGrafanas=1,skipServices=1,skipIngress=1,skipStatic=1,skipAPI=1,skipNamespaces=1,patroniRetryTimeout=120,patroniTtl=120,postgresMaxConn=16,postgresMaxParallelWorkers=2,postgresMaxReplicationSlots=2,postgresMaxTempFile=1GB,postgresMaxWalSenders=2,postgresMaxWorkerProcesses=2,postgresSharedBuffers=1280MB,postgresStorageSize=30Gi,postgresWalBuffers=64MB,postgresWalKeepSegments=5,postgresWorkMem=256MB,requestsPostgresCPU=400m,requestsPostgresMemory=512Mi`.
- Shell into the patroni master pod (after all 4 patroni nodes are in `Running` state: `k get po -n devstats-test | grep devstats-postgres-`): `k exec -n devstats-test -it devstats-postgres-0 -- /bin/bash`:
  - Run: `patronictl list` to see patroni cluster state.
  - Tweak patroni: `curl -s -XPATCH -d '{"loop_wait": "60", "postgresql": {"parameters": {"shared_buffers": "1280MB", "max_parallel_workers_per_gather": "2", "max_connections": "16", "max_wal_size": "1GB", "effective_cache_size": "1GB", "maintenance_work_mem": "256MB", "checkpoint_completion_target": "0.9", "wal_buffers": "64MB", "max_worker_processes": "2", "max_parallel_workers": "2", "temp_file_limit": "1GB", "hot_standby": "on", "wal_log_hints": "on", "wal_keep_segments": "5", "wal_level": "hot_standby", "max_wal_senders": "2", "max_replication_slots": "2"}, "use_pg_rewind": true}}' http://localhost:8008/config | jq .`.
  - `patronictl restart --force devstats-postgres`.
  - `patronictl show-config` to confirm config.
- Check patroni logs: `k logs -n devstats-test -f devstats-postgres-N`, N=0,1,2,3.
- Install static pages handlers: `helm install devstats-test-statics ./devstats-helm --set skipSecrets=1,skipPVs=1,skipBackupsPV=1,skipVacuum=1,skipBackups=1,skipBootstrap=1,skipProvisions=1,skipCrons=1,skipAffiliations=1,skipGrafanas=1,skipServices=1,skipPostgres=1,skipIngress=1,skipAPI=1,skipNamespaces=1,indexStaticsFrom=0,indexStaticsTo=1,requestsStaticsCPU=50m,requestsStaticsMemory=64Mi`.
- Install ingress: `helm install devstats-test-ingress ./devstats-helm --set skipSecrets=1,skipPVs=1,skipBackupsPV=1,skipVacuum=1,skipBackups=1,skipBootstrap=1,skipProvisions=1,skipCrons=1,skipAffiliations=1,skipGrafanas=1,skipServices=1,skipPostgres=1,skipStatic=1,skipAPI=1,skipNamespaces=1,indexDomainsFrom=0,indexDomainsTo=1,ingressClass=nginx-test,sslEnv=test`.
- Provision initial logs database and infra: `helm install devstats-test-bootstrap ./devstats-helm --set skipSecrets=1,skipPVs=1,skipBackupsPV=1,skipVacuum=1,skipBackups=1,skipProvisions=1,skipCrons=1,skipAffiliations=1,skipGrafanas=1,skipServices=1,skipPostgres=1,skipIngress=1,skipStatic=1,skipAPI=1,skipNamespaces=1,requestsBootstrapCPU=100m,requestsBootstrapMemory=128Mi`.
- Delete finished bootstrap pod (when in `Completed` state): `k delete po devstats-provision-bootstrap`.
- Create backups cron job: `helm install devstats-test-backups ./devstats-helm --set skipSecrets=1,skipPVs=1,skipBackupsPV=1,skipVacuum=1,skipBootstrap=1,skipProvisions=1,skipCrons=1,skipAffiliations=1,skipGrafanas=1,skipServices=1,skipPostgres=1,skipIngress=1,skipStatic=1,skipAPI=1,skipNamespaces=1,backupsPVSize=2Gi,requestsBackupsCPU=100m,requestsBackupsMemory=128Mi`.
- Install KEDA and SMI projects: `helm install devstats-test-keda-smi ./devstats-helm --set skipSecrets=1,skipPVs=1,skipBackupsPV=1,skipVacuum=1,skipBackups=1,skipBootstrap=1,indexProvisionsFrom=70,indexProvisionsTo=72,indexCronsFrom=70,indexCronsTo=72,indexGrafanasFrom=70,indexGrafanasTo=72,indexServicesFrom=70,indexServicesTo=72,indexAffiliationsFrom=70,indexAffiliationsTo=72,skipPostgres=1,skipIngress=1,skipStatic=1,skipAPI=1,skipNamespaces=1,skipAddAll=1,requestsAffiliationsCPU=100m,requestsAffiliationsMemory=128Mi,requestsCronsCPU=100m,requestsCronsMemory=128Mi,requestsGrafanasCPU=100m,requestsGrafanasMemory=128Mi,requestsProvisionsCPU=500m,requestsProvisionsMemory=256Mi`.
- Or restore SMI from the backup: `helm install devstats-test-restore-smi ./devstats-helm --set skipSecrets=1,skipPVs=1,skipBackupsPV=1,skipVacuum=1,skipBackups=1,skipBootstrap=1,indexProvisionsFrom=71,indexProvisionsTo=72,indexCronsFrom=71,indexCronsTo=72,indexGrafanasFrom=71,indexGrafanasTo=72,indexServicesFrom=71,indexServicesTo=72,indexAffiliationsFrom=71,indexAffiliationsTo=72,skipPostgres=1,skipIngress=1,skipStatic=1,skipAPI=1,skipNamespaces=1,skipAddAll=1,requestsAffiliationsCPU=100m,requestsAffiliationsMemory=128Mi,requestsCronsCPU=100m,requestsCronsMemory=128Mi,requestsGrafanasCPU=100m,requestsGrafanasMemory=128Mi,requestsProvisionsCPU=800m,requestsProvisionsMemory=512Mi,provisionCommand='devstats-helm/restore.sh',restoreFrom='https://teststats.cncf.io/backups/'`.
- Install reporting pod: `helm install devstats-test-reports ./devstats-helm --set skipSecrets=1,skipPVs=1,skipBackupsPV=1,skipVacuum=1,skipBackups=1,skipBootstrap=1,skipProvisions=1,skipCrons=1,skipAffiliations=1,skipGrafanas=1,skipServices=1,skipPostgres=1,skipIngress=1,skipStatic=1,skipAPI=1,skipNamespaces=1,reportsPod=1,requestsReportsCPU=100m,requestsReportsMemory=128Mi`.
- Shell into reporting pod: `k exec -it devstats-reports -- /bin/bash`, do some operations on the SMI db: `db.sh psql smi`, run some report: `PG_DB=smi ./affs/committers.sh`. Exit `exit`.
- Delete no more needed reporting pod: `helm delete devstats-test-reports`.
- This is a smallest VM DevStats demo, it will be too slow to be usable, it's just a POC.
