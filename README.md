### Versioner av linux som stöds
* Ubuntu 16.04 ->
* Debian 9 ->
* CentOS 7 ->
* RHEL 7 ->
* Fedora 25 ->
* HypriotOS v1.0.1 ->

### Minimum kvar på hårdvara
2 CPU(4vCPU) och 2 GB RAM. Masternoderna om 3 behöver ligga på samma subnät för att ha och quorum ska fungera optimalt. Alla virtuella eller fysiska servrar behöver ha unika hostnamn, MAC-adresser och uuid(machine id). Alla servrar behöver ha statisk ip och möjlighet att slå upp dns.

#### Swap måste vara av på master/worker-noder
````
swapoff -a
cat /etc/fstab (ta bort monteringen av swap)
````

### IP inställningar(kan se annorlunda ut om netplan används)
````
vi /etc/network/interfaces
# The primary network interface
allow-hotplug eth0
iface eth0 inet static
        address 10.255.13.12/24
        gateway 10.255.13.1
        dns-nameservers 10.255.20.2 10.255.20.3
        dns-search k8s.vmar.se
`````

#### hostfil eller dns-uppslag (rekommenderat att lägga till namn i hostfil)
````
cat /etc/hosts (denna fil behöver vara samma på alla servrar i hela klustret helst, masterservrar är viktigast.)
127.0.0.1	localhost

172.18.251.41	kubernetes-master-1
172.18.251.42	kubernetes-master-2
172.18.251.43	kubernetes-master-3
````

## Skapa ssh public/private nyckel (copy this to all hosts if running multimaster or ansible)
````
ssh-keygen -t rsa -b 4096 -f $(users | sed 's/[^[:alnum:]\t]//g')_$(hostname | sed 's/[^[:alnum:]\t]//g')_$(date '+%Y-%m-%d')_id_rsa -q -P ''

mkdir .ssh

cp *_id_rsa.pub ~/.ssh/id_rsa.pub
cp *_id_rsa ~/.ssh/id_rsa

root@upp-k8s-01:~# ssh-copy-id localhost
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host 'localhost (127.0.0.1)' can't be established.
ECDSA key fingerprint is SHA256:AVTykMVFTt4SKcaMz5aifvruRcRo6oEDch70PQlZxIM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@localhost's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'localhost'"
and check to make sure that only the key(s) you wanted were added.

````

### Kopiera sedan nyckeln till alla noder som finns i klustret.
````
scp -r .ssh <other-node>:
````

#### lägg till i sysctl och modules ( tillåter kubernetes att se in i bryggor)
````
# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1 # needed for kubelet
net.bridge.bridge-nf-call-ip6tables = 1 # needed for kubelet
net.ipv4.ip_nonlocal_bind = 1 # only for haproxy
EOF
sudo sysctl --system
````

## Install all prereq
````
apt install -y ifupdown net-tools iproute2 nmon htop iftop ipvsadm ipset vim ethtool bridge-utils python3-pip git snmp snmpd apt-transport-https ca-certificates curl software-properties-common lsof dnsutils cloud-utils tmux ncdu tcpdump pwgen jq open-vm-tools bash-completion mtr lvm2 lftp screen wget vim nmap qemu-guest-agent socat conntrack ebtables resolvconf
````

#### Stäng av selinux(OM REDHAT ELLER CENTOS) eller lägg det i läge "permissive" för att senare lägga till undantag
````

sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sestatus
vi /etc/selinux/config (permissive eller disabled)
setenforce 0

starta om efter detta är gjort.

````


### Brandväggsportar

eller stäng av firewalld (Oftast REDHAT/CENTOS)

````
systemctl disable ufw
systemctl disable firewalld
reboot
````

#### Masternoder
| Protokoll | Riktning | Portar | Syfte | Användning |
|-----------|----------|--------|-------|------------|
|TCP|Inkommande|6443|kube-api|Alla|
|TCP|Inkommande|2379-2380|etcd server client/api|kube-api, etcd|
|TCP|Inkommande|10250|kubelet-api|self, controlplane|
|TCP|Inkommande|10251|kube-scheduler|self|
|TCP|Inkommande|10252|kube-controller-manager|self|

#### Workernoder
| Protokoll | Riktning | Portar | Syfte | Användning |
|-----------|----------|--------|-------|------------|
|TCP|Inkommande|10250|kubelet-api|self, controlplane|
|TCP|Inkommande|30000-32767|nodePort,svc|all|

## Install containerd.io
````
# Add docker repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt install containerd.io

mkdir -p /etc/containerd
cat /etc/containerd/config.toml
version = 2
root = "/var/lib/containerd"
state = "/run/containerd"
disabled_plugins = []
required_plugins = ["io.containerd.grpc.v1.cri"]
subreaper= true
oom_score = -999

[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    enable_selinux = false
  [plugins."io.containerd.runtime.v1.linux"]
    shim = "containerd-shim"
    systemd_cgroup = true
    runtime = "runc"
    no_shim = false
    shim_debug = false
````

## Setup keepalived
````
apt install keepalived -y

cat /etc/keepalived/keepalived.conf

! Configuration File for keepalived

global_defs {
  # Keepalived process identifier
  enable_script_security
  script_user root
}

vrrp_script chk_haproxy {
  script "/usr/bin/killall -0 haproxy"
  interval 2
  weight 2
}

vrrp_instance kubernetes-api {
  interface <HUVUD NIC, ETC ETH0>
  state MASTER
  virtual_router_id 10
  priority 100
  virtual_ipaddress {
       <VIP IP>
  }
  track_script {
       chk_haproxy
  }
}

systemctl enable keepalived
systemctl start keepalived
systemctl status keepalived
````

## Install haproxy
````
apt install -y haproxy

cat /etc/haproxy/haproxy.cfg

global
	....
defaults
	....

listen stats
	bind <VIP IP>:9999
	option http-use-htx
	http-request use-service prometheus-exporter if { path /metrics }
	stats enable
	stats uri /stats
	stats refresh 5s
	
frontend kubernetes
	bind <VIP IP>:8443
	option tcplog
	mode tcp
	default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
	mode tcp
	balance roundrobin
	option tcp-check
	server <NODNAMN> <NOD IP>:6443 check fall 3 rise 2
	server <NODNAMN> <NOD IP>:6443 check fall 3 rise 2
	server <NODNAMN> <NOD IP>:6443 check fall 3 rise 2
  
  systemctl enable haproxy
  systemctl start haproxy
  systemctl status haproxy
  
````

### Installera kubernetes
````
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt update

############# FÖR ATT VÄLJA SÄRSKILDA PAKET ############
https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-arm64/Packages
apt install kubeadm=1.19.11-00 kubelet=1.19.11-00 kubectl=1.19.11-00
apt install kubeadm=1.18.9-00 kubelet=1.18.9-00 kubectl=1.18.9-00
########################################################

apt install -y kubeadm kubectl kubelet
apt-mark hold kubeadm kubectl kubelet
````


### Initiera kubernetes cluster
Det finns flera olika sätt att initiera klustret. Många beror på vilket nätverksplugin som används. Nedan är gjort för "CANAL", det är en blandning av flannel och project calico.

Flannel har ingen säkerhet, den skapar endast ett VXLAN mellan alla masternoder. Project Calico ligger med för att säkra trafiken mellan "PODs". Canal är en bra blandning av hastighet, enkelhet och säkerhet med Calico.

* --control-plane-endpoint <- Den gemensamma ip som används av klienter
* --upload-certs <- laddar upp certen som skapas på första master till de andra masternoderna.
* --pod-network-cidr <- Detta är ett nät som kommer ligga på VXLAN och hanteras 100% av kubernetes. Det kommer aldrig att störa andra nät i befintlig infrastruktur.
* --service-cidr <- Ändra nätet där interna lastbalanserare körs. Standard är 10.96.0.0/12 

https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/

````
sysctl -w net.ipv4.ip_forward=1 ## ONLY FIRST TIME OR WHEN JOINING MORE NODES
kubeadm init --control-plane-endpoint="keepalived-vip-ip:8443" --upload-certs --pod-network-cidr="10.244.0.0/16"
````
#### När klustret är installerat kommer du få en text hur du använder kubectl och tar med nya masternoder. Spar denna någonstans
För att skapa ett nytt kommando "--print-join-command"
````
kubeadm token create --control-plane --print-join-command    <- För masternoder
kubeadm token create --print-join-command      <- För workernoder
````

#### Sätt namn på nod
````
kubectl label node <nodnamn> node-role.kubernetes.io/worker=worker
````

#### För att kunna använda "kubectl"
````
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
````

#### För att kunna köra "deployments" på masternoder om du inte har några workers
````
kubectl taint nodes --all node-role.kubernetes.io/master-
````
#### Applicera en nätverksplugin (CANAL)
````
kubectl apply -fhttps://docs.projectcalico.org/manifests/canal.yaml

# Fix ipvs stuffs
kubectl edit configmap kube-proxy -n kube-system 
change strictARP: false -> true
change mode: "" > "ipvs"
kubectl -n kube-system rollout restart daemonset kube-proxy
````

### Kontrollera att allt fungerar som det ska
````
kubectl get all -A
kubectl get pods -n kube-system -o wide
````

### Installera helm
````
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
````

### Installera metallb
````
kubectl create ns metallb-system
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install lb bitnami/metallb --namespace metallb-system

cat metallb.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: lb-metallb-config
data:
  config: |
    address-pools:
    - name: internal
      protocol: layer2
      addresses:
      - 172.18.251.44-172.18.251.49
    - name: traefik
      protocol: layer2
      addresses:
      - 172.18.251.50/32  
````

### Installera longhorn
````
git clone https://github.com/longhorn/longhorn
kubectl create ns longhorn-system
helm install longhorn ./longhorn/chart/ --namespace longhorn-system
````

### Installera traefik
````
helm repo add traefik https://helm.traefik.io/traefik
kubectl create ns traefik-system
helm install traefik traefik/traefik --namespace traefik-system
````

### Installera rancher
````
helm repo add jetstack https://charts.jetstack.io
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

kubectl create namespace cert-manager
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set installCRDs=true

kubectl create namespace cattle-system
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=<hostnamn>
````

