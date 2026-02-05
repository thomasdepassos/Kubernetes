# Guide d'Installation : Cluster Kubernetes HA
**Architecture : 3 Masters, 2 Workers | VIP : 10.0.3.110**

Ce guide détaille la mise en place d'un cluster Kubernetes hautement disponible en utilisant `kubeadm`. L'équilibrage de charge pour l'API server est géré par Keepalived et HAProxy sur les nœuds Masters.

## 1. Inventaire des Nœuds

| Rôle | Nom du Nœud | IP Physique |
| :--- | :--- | :--- |
| **VIP (Load Balancer)** | **Kubernetes Endpoint** | **10.0.3.110** |
| Master 1 | `master01` | 10.0.3.100 |
| Master 2 | `master02` | 10.0.3.101 |
| Master 3 | `master03` | 10.0.3.102 |
| Worker 1 | `worker01` | 10.0.3.21 |
| Worker 2 | `worker02` | 10.0.3.22 |


## Configuration Commune (Tous les nœuds)
### Désactivation du Swap & Modules Noyau
```bash
# Désactiver le swap
sudo swapoff -a
sudo systemctl mask dev-nvme0n1p3.swap 

# Charger les modules requis
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Configurer les paramètres sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

## Installation Containerd (Runtime)
```bash
sudo apt update && sudo apt install -y containerd

sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd 
sudo systemctl enable containerd
```
### Installation des outils Kubernetes
```bash
sudo apt install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
sudo systemctl enable kubelet
```

## 2. Étape Préliminaire : Load Balancer (Masters uniquement)
Installez **HAProxy** et **Keepalived** sur les 3 nœuds Masters pour gérer l'IP virtuelle (VIP) et la redirection du trafic vers l'API Server.

### Installation des paquets

```bash
sudo apt install -y keepalived haproxy
```

## Configuration HAProxy `/etc/haproxy/haproxy.cfg`

```bash
frontend k8s-api
    bind 10.0.3.110:8443
    mode tcp
    option tcplog
    default_backend k8s-masters

backend k8s-masters
    mode tcp
    option tcp-check
    balance roundrobin
    server master01 10.0.3.100:6443 check
    server master02 10.0.3.101:6443 check
    server master03 10.0.3.102:6443 check
```

## Configuration Keepalived `/etc/keepalived/keepalived.conf`

```bash
vrrp_instance VI_1 {
    state MASTER
    interface ens3
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass toto
    }
    virtual_ipaddress {
        10.0.3.110
    }
}
```
```bash
systemctl restart keepalived haproxy
```

### Prérequis de longhorn
```bash
sudo apt install -y open-iscsi nfs-common
sudo systemctl enable --now iscsid
```

### Initialisation du Cluster (Sur Master 1 : 10.0.3.100)

```bash
sudo kubeadm init \
  --control-plane-endpoint "10.0.3.110:8443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16
  ```

### Ajouter les masters au cluster

#### Sur les masters : 
```bash
kubeadm join 10.0.3.110:8443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane --certificate-key <CERT_KEY>
  ```

  ### Ajouter les Workers dans le cluster
  #### Sur les Workers : 

  ```bash
  kubeadm join 10.0.3.110:8443 --token <TOKEN> \
    --discovery-token-ca-cert-hash sha256:<TON_HASH>
```

### Set up kubectl sur les 3 Masters

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Installation du CNI (Flannel)

```bash
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

### Stockage Haute Disponibilité (Longhorn)

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/deploy/longhorn.yaml
```

### Déploiement WordPress HA (avec PVC Longhorn)

```bash
kubectl apply -f mysql.yml
kubectl apply -f wordpress.yml
```

