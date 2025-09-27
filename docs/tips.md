# Conteinerd

Iniciar
``` sudo systemctl start containerd ```

Verificar quais pods estão rodando
``` sudo ctr containers list ```
``` sudo ctr tasks list ```

# Kubeadm

Remover cluster
sudo kubeadm -f reset

Iniciar cluster
sudo kubeadm init --control-plane-endpoint "nome-da-maquina-dns" --pod-network-cidr=10.244.0.0/16

# Kubectl

Verificar nodes
kubectl get nodes

 kubectl describe node <node>

# CNI

Instalação Antrea
kubectl apply -f https://github.com/antrea-io/antrea/releases/download/v2.2.2/antrea.yml
sudo mkdir -p /usr/lib/cni
sudo ln -s /opt/cni/bin/* /usr/lib/cni/