# Configuração de Cluster

## Objetivo

Instalar e configurar cluster Kubernetes funcional, atendendo as boas práticas para ambiente de produção e seguro com as características descritas abaixo

Cluster bare-metal, hospedado em VM's própria em ambiente de nuvem privada, sem HA configurado, com 2 nós que hospederão serviços de control-plane (componenentes internos do cluster e banco dados etcd) e worker nodes (pods de aplicações)

## Nós
- VM's de nuvem privada com 2 vCpus e 8GB de RAM e disco de 100GB, com Ubuntu Server 24 com SO.
- VM's devem ter nomes definidos seguindo namespace padrão emp-k8s-01.empresa.com.br e emp-k8s-02.empresa.com.br (analisar necessidade de configuração no DNS - fora da rede)

### Visão macro

Componentes principais:
| Função | Componente | Justificativa |
|-|-|-|
| Container Runtime | containerd | Padrão / amplo uso no mercado |
| Ferramenta de configuração/manutenção | kubeadm | Padrão, permite customização do cluster |
| CNI | Antrea | Não possui issues com MetalLB |
| Load Balancer | Metal LB | Provisionamento de IP's externos |
| API Gateway | Istio | Padrão / amplo uso no mercado |
| Gerenciador de Certificados | cert-manager | padrão |


## Pré-requisitos e instalação de ferramentas

 Documentação básica de instalação

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### Principais passos

### Desabilitar SWAP

Deve-se desabilitar o SWAP do nó para evitar falhas no kubelet (configuração padrão)

```
sudo swapoff -a
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
```

### Configurações de Kernel / Iptables

Garantir carregamento dos módulos overlay e br_netfilter no boot / carregar os módulos imediatamente

overlay: Utilizado pelo driver de armazenamento OverlayFS, que é comum em ambientes de contêineres.

br_netfilter: Permite que o tráfego de rede em interfaces de ponte (bridge) seja inspecionado por ferramentas como iptables, essencial para o funcionamento correto do Kubernetes.

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

```

Configurações de rede

| Parâmetro |	Valor recomendado |	O que faz / por que é necessário |
|-|-|-|
| net.bridge.bridge-nf-call-iptables = 1 | 1 | Faz com que o tráfego de ponte (bridge) fique sujeito às regras do iptables. Sem isso, o tráfego entre | contêineres pode “pular” regras de iptables se estiver em ponte.
| net.bridge.bridge-nf-call-ip6tables = 1 |	1 |	Versão IPv6 do anterior
| net.ipv4.ip_forward = 1 |	1 |	Habilita forwarding/IP routing no kernel, necessário para roteamento entre interfaces / sub-redes

```
sudo tee /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

```

### Instalar containerd

```

sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

```



### Instalação de ferramentas e Kubelet

```
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

```

## Inicialização do Cluster e configuração dos componentes

### Iniciar cluster

Normalmente o proprio kubeadm gera um CA interna para comunicação segura entre os componentes. Verificar se há necessidade de gerar os certificados a partir de CA própria como na doc https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#using-your-own-certificates

Nesse exemplo, pode-se copiar a CA para /etc/kubernetes/pki (ca.crt e ca.key), que na inicialização o kubeadm usará está CA para assinar os demais certificados

```
sudo kubeadm init --control-plane-endpoint "nome-da-maquina-dns" --pod-network-cidr=10.244.0.0/16
```

### Criar usuário 

```
sudo adduser k8suser
sudo usermod -aG sudo k8suser   -- Verificar segurança
su - k8suser 

mkdir -p $HOME/.kube

sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

## Instalação de componentes no Cluster

### Instalação do CNI

```
kubectl apply -f https://github.com/antrea-io/antrea/releases/download/v2.2.2/antrea.yml
sudo mkdir -p /usr/lib/cni
sudo ln -s /opt/cni/bin/* /usr/lib/cni/
```

### MetalLB

https://metallb.io/installation/index.html

https://metallb.io/configuration/_advanced_ipaddresspool_configuration/

Exemplo de configuração:

```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.15.240-192.168.15.250   # Ajuste para um intervalo livre na sua rede local

```

### Istio

https://istio.io/latest/docs/ambient/getting-started/

https://istio.io/latest/docs/setup/install/helm/

### Cert-manager

### LongHorn


## Adicionar novos nós

## Testes

### Incluir deployments e services de aplicações

## Dúvidas

- Como rotacionar certificados (e renovar) ?
- Como fazer o backup do etcd ?
- Como garantir HA
