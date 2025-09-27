
# Kubernetes POC

4 vCPU + 8GB RAM

=> Geração de certificados a partir do AD

KubeAdmin
    Certificado para ele

Instalação pura do Linux
mTLS => chave cliente OAuth
    Comunicação entre cluster

CA Becomex
Host 
(TLS)

Containerd
MetalLB (Instalação em cada nó + driver via Helm Chart)
LongHorn

system.d

Istio (NEST -> CA Intermediária) => TLS
Criar novo domínio k8s.empresa.com.br


Rancher

Usar OAuth para autenticação

## Passo a passo

TI:
- Provisionamento de VM's
- Atribuição de nome para os nós DNS: k8s-01.corporacao.corp

TI/Arq:
- Gerar CA intermediária para provisionamento de certificados

Engenheria

1. Criação de usuário para cluster
2. 


## Importante

## SSH
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh


### Desativar SWAP
sudo swapoff -a
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab

### Kernel params: iptables sobre bridge
cat <<'SYSCTL' | sudo tee /etc/sysctl.d/99-k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
SYSCTL
sudo modprobe br_netfilter
sudo sysctl --system

### Instalar containerd
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

### Usar SystemdCgroup=true (recomendado pelo kubeadm)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd

### Instalar kubeadm/kubelet/kubectl

sudo apt-get update

sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

### Inicializar o cluster

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=v1.30.0

### Criar usuário 

sudo adduser k8suser
sudo usermod -aG sudo k8suser   # se quiser privilégios sudo
su - k8suser

mkdir -p $HOME/.kube

sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


### Instalar o CNI (Rede de pods)

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml

sudo mkdir -p /usr/lib/cni
sudo ln -s /opt/cni/bin/* /usr/lib/cni/


## Dia-dia

kubectl get pods -A

## Glossário

Calico = Serviço de rede entre dos containers
Containerd = Container runtime (CRI), equivalente ao Docker. Instalado em cada nó

## Configuração
/etc/kubernetes/admin.conf
/etc/kubernetes/manifests/kube-apiserver.yaml

/etc/kubernetes/manifests/

## Logs
sudo journalctl -u kubelet -f

## Reiniciar serviços
sudo systemctl restart kubelet
sudo systemctl restart containerd


certificados com SAN (Subject Alternative Name)



## Explicações

Service 
O service como se fosse o porteiro do Pods, quem garante acesso. Existem 3 tipo de services

ClusterIP - Pod só fica acessivel dentro do cluster
NodePort - Expoe o pod numa porta especifica
LoadBalancer - Cria um LB externo que distribui o trafego entre os pods





# VIA DOC

## Instalar Kubectl
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

## Pre-requisitos
https://kubernetes.io/docs/setup/production-environment/container-runtimes/

## Contem Nerd
https://github.com/containerd/containerd/blob/main/docs/getting-started.md

## Instalar kubeadm
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/