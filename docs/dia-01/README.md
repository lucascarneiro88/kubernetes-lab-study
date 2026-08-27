# Laboratório Kubernetes — Preparação e criação de cluster kubeadm de 3 nós

## 1. Objetivo desta etapa

O objetivo desta etapa foi preparar três máquinas virtuais Ubuntu para formar um cluster Kubernetes utilizando:

* `kubeadm` para criação e ingresso no cluster;
* `kubelet` para executar e gerenciar os Pods em cada nó;
* `kubectl` para administrar o cluster;
* `containerd` como runtime de containers;
* `Flannel` como rede de Pods;
* 1 nó control-plane;
* 2 nós workers.

A topologia utilizada foi:

| Máquina     | Função        | IP Host-Only  |
| ----------- | ------------- | ------------- |
| k8s-master  | Control Plane | 192.168.56.10 |
| k8s-worker1 | Worker        | 192.168.56.11 |
| k8s-worker2 | Worker        | 192.168.56.12 |

> Observação: o Kubernetes acabou identificando `10.0.2.15` como `INTERNAL-IP` nos três nós. Isso indica que ele selecionou a interface NAT como endereço interno. O cluster ficou funcional e os três nós chegaram a `Ready`, mas essa configuração merece ser estudada posteriormente.

---

# 2. Desligamento e inicialização das VMs

As máquinas estavam inicialmente desligadas.

No PowerShell, dentro do diretório do laboratório:

```powershell
vagrant status
```

O comando mostra o estado das máquinas administradas pelo Vagrant.

Para ligar todas:

```powershell
vagrant up
```

Depois:

```powershell
vagrant status
```

Resultado esperado:

```text
k8s-master    running
k8s-worker1   running
k8s-worker2   running
```

Para acessar uma máquina:

```powershell
vagrant ssh k8s-master
```

ou:

```powershell
vagrant ssh k8s-worker1
```

ou:

```powershell
vagrant ssh k8s-worker2
```

---

# 3. Conferência da rede da master

Na `k8s-master` executamos:

```bash
hostname
```

Esse comando mostra o nome da máquina.

Resultado:

```text
k8s-master
```

Depois:

```bash
hostname -I
```

Esse comando mostra os endereços IP associados às interfaces da máquina.

Foi identificado:

```text
10.0.2.15
192.168.56.10
```

Também verificamos especificamente a interface Host-Only:

```bash
ip a | grep 192.168.56
```

Resultado:

```text
inet 192.168.56.10/24 ...
```

Isso confirmou que a interface Host-Only estava configurada com o endereço esperado.

---

# 4. Desativação do Swap

O Kubernetes tradicionalmente requer que o Swap esteja desativado para evitar problemas de gerenciamento de memória e comportamento inconsistente do kubelet.

Primeiro verificamos:

```bash
swapon --show
```

Se não aparecer nada, significa que não há Swap ativo.

Também verificamos:

```bash
grep swap /etc/fstab
```

Se não aparecer nenhuma linha de Swap, não existe uma configuração de Swap persistente no `/etc/fstab`.

Quando necessário, os comandos utilizados para desativar seriam:

```bash
sudo swapoff -a
```

O `swapoff -a` desativa todas as áreas de Swap atualmente ativas.

Depois:

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Esse comando procura linhas relacionadas a Swap no `/etc/fstab` e coloca `#` no início delas, transformando-as em comentários.

Assim, o Swap não volta automaticamente após reiniciar a máquina.

No nosso laboratório, as verificações posteriores mostraram que não havia Swap ativo.

---

# 5. Módulos do kernel necessários

Foram carregados dois módulos:

```bash
sudo modprobe overlay
```

e:

```bash
sudo modprobe br_netfilter
```

## overlay

O módulo `overlay` fornece o OverlayFS, utilizado pelo runtime de containers para montar camadas de sistemas de arquivos.

Containers utilizam frequentemente um sistema de arquivos formado por várias camadas. O OverlayFS permite combinar essas camadas de maneira eficiente.

O `containerd` também utiliza esse mecanismo.

## br_netfilter

O módulo:

```text
br_netfilter
```

permite que o tráfego que passa por bridges Linux seja processado pelas regras de filtragem de rede do kernel.

Isso é importante em ambientes Kubernetes porque containers e Pods frequentemente utilizam interfaces virtuais e bridges.

---

# 6. Tornando os módulos persistentes

Carregar o módulo com `modprobe` funciona apenas para o sistema em execução.

Para garantir que os módulos sejam carregados novamente após um reboot, criamos:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

O arquivo:

```text
/etc/modules-load.d/k8s.conf
```

passou a conter:

```text
overlay
br_netfilter
```

Podemos conferir com:

```bash
cat /etc/modules-load.d/k8s.conf
```

Assim, o sistema sabe que esses módulos devem ser carregados durante a inicialização.

---

# 7. Configuração de rede do kernel

Criamos:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Esse arquivo contém três configurações importantes.

## net.bridge.bridge-nf-call-iptables

```text
net.bridge.bridge-nf-call-iptables = 1
```

Permite que tráfego de bridges seja processado pelas regras do iptables.

Isso é importante para o funcionamento das regras de rede utilizadas pelo Kubernetes.

## net.bridge.bridge-nf-call-ip6tables

```text
net.bridge.bridge-nf-call-ip6tables = 1
```

Faz a mesma coisa para tráfego IPv6.

## net.ipv4.ip_forward

```text
net.ipv4.ip_forward = 1
```

Ativa o encaminhamento de pacotes IPv4.

Isso permite que a máquina encaminhe tráfego entre interfaces, algo fundamental para a comunicação entre diferentes redes e para o funcionamento da rede de Pods.

---

# 8. Aplicação das configurações

Depois executamos:

```bash
sudo sysctl --system
```

O `sysctl` administra parâmetros do kernel.

A opção:

```text
--system
```

faz o sistema carregar as configurações dos arquivos de configuração do `sysctl`.

Entre as configurações aplicadas apareceu:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

Também apareceram algumas mensagens:

```text
sysctl: setting key ... Invalid argument
```

Essas mensagens vieram de outras configurações existentes no sistema, não das três configurações que adicionamos para o Kubernetes.

As configurações do arquivo:

```text
/etc/sysctl.d/k8s.conf
```

foram aplicadas corretamente.

---

# 9. Instalação do containerd

O Kubernetes precisa de um runtime de containers.

Escolhemos:

```text
containerd
```

Na master verificamos inicialmente:

```bash
containerd --version
```

Como ainda não estava instalado, o sistema informou:

```text
Command 'containerd' not found
```

Então instalamos:

```bash
sudo apt-get update
```

O comando atualiza a lista de pacotes disponíveis nos repositórios configurados.

Depois:

```bash
sudo apt-get install -y containerd
```

Também foi instalado o `runc`, que é utilizado pelo containerd para executar containers.

Depois verificamos:

```bash
systemctl status containerd --no-pager
```

O resultado mostrou:

```text
Active: active (running)
```

Isso significa que o serviço estava funcionando.

---

# 10. Configuração do containerd para Kubernetes

Criamos o diretório de configuração:

```bash
sudo mkdir -p /etc/containerd
```

Depois geramos uma configuração padrão:

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Esse comando solicita ao containerd sua configuração padrão e grava o resultado em:

```text
/etc/containerd/config.toml
```

Verificamos:

```bash
grep -n "SystemdCgroup" /etc/containerd/config.toml
```

Inicialmente estava:

```text
SystemdCgroup = false
```

Alteramos para:

```text
SystemdCgroup = true
```

com:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

## Por que SystemdCgroup = true?

O Kubernetes utiliza cgroups para controlar recursos dos containers, como:

* memória;
* CPU;
* processos.

O sistema operacional Ubuntu utiliza o systemd como gerenciador de serviços e cgroups.

Configurar:

```text
SystemdCgroup = true
```

faz o containerd utilizar o systemd como driver de cgroup, mantendo o runtime alinhado com o gerenciamento de recursos utilizado pelo sistema.

Depois conferimos novamente:

```bash
grep -n "SystemdCgroup" /etc/containerd/config.toml
```

Obtivemos:

```text
SystemdCgroup = true
```

Finalmente reiniciamos:

```bash
sudo systemctl restart containerd
```

E verificamos:

```bash
systemctl status containerd --no-pager
```

Resultado:

```text
Active: active (running)
```

Também verificamos:

```bash
systemctl is-enabled containerd
```

Resultado:

```text
enabled
```

Isso significa que o containerd está configurado para iniciar automaticamente junto com o sistema.

---

# 11. Repetição da preparação nos workers

Os mesmos conceitos foram aplicados ao:

```text
k8s-worker1
```

e:

```text
k8s-worker2
```

Em cada worker foram configurados:

* Swap;
* `overlay`;
* `br_netfilter`;
* `/etc/modules-load.d/k8s.conf`;
* `/etc/sysctl.d/k8s.conf`;
* `containerd`;
* `SystemdCgroup = true`;
* containerd habilitado e iniciado.

Também verificamos o serviço:

```bash
systemctl status containerd --no-pager
```

com resultado:

```text
Active: active (running)
```

---

# 12. Pequenos erros durante o laboratório

Houve alguns erros de digitação que não afetaram o laboratório.

## ca-certicates

Foi digitado:

```bash
sudo apt-get install -y apt-transport-https ca-certicates curl gpg
```

O correto é:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

O nome correto do pacote é:

```text
ca-certificates
```

e não:

```text
ca-certicates
```

Depois corrigimos e o comando funcionou.

---

## conatinerd

Também foi digitado:

```bash
systemctl status conatinerd --no-pager
```

O correto é:

```bash
systemctl status containerd --no-pager
```

O primeiro comando apenas falhou porque o nome do serviço foi digitado incorretamente.

---

# 13. Repositório oficial do Kubernetes

Configuramos o repositório de pacotes da série Kubernetes 1.33.

Primeiro:

```bash
sudo apt-get update
```

Depois instalamos os pacotes necessários:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

Criamos o diretório para as chaves:

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
```

Baixamos a chave do repositório:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Essa chave permite verificar criptograficamente os pacotes recebidos do repositório.

Depois criamos:

```text
/etc/apt/sources.list.d/kubernetes.list
```

com:

```text
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /
```

Atualizamos novamente:

```bash
sudo apt-get update
```

O repositório Kubernetes foi reconhecido corretamente.

---

# 14. Instalação dos componentes Kubernetes

Instalamos:

```bash
sudo apt-get install -y kubelet kubeadm kubectl
```

Foram instalados:

* `kubeadm`;
* `kubectl`;
* `kubelet`;
* `kubernetes-cni`.

## kubeadm

É utilizado para criar e inicializar o cluster e para adicionar nós existentes ao cluster.

## kubelet

É o agente que roda em cada máquina Kubernetes.

Ele recebe instruções do control plane e garante que os Pods atribuídos ao nó estejam executando.

## kubectl

É a ferramenta de linha de comando utilizada para conversar com a API do Kubernetes.

## kubernetes-cni

Fornece componentes relacionados à interface de rede utilizada pelos Pods.

---

# 15. Bloqueio das versões

Nos workers executamos:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Isso impede que esses pacotes sejam atualizados automaticamente pelo gerenciamento normal de pacotes.

A ideia é evitar que uma atualização automática altere a versão do Kubernetes sem planejamento.

---

# 16. Inicialização do kubelet

Em cada máquina executamos:

```bash
sudo systemctl enable --now kubelet
```

O comando faz duas coisas:

```text
enable
```

faz o serviço iniciar automaticamente com o sistema.

```text
--now
```

também inicia o serviço imediatamente.

Antes do `kubeadm init`, o kubelet pode aparecer sem conseguir executar sua função completa, porque o cluster ainda não foi criado.

Isso é normal.

---

# 17. Verificação das versões

Na master verificamos:

```bash
kubeadm version
```

Resultado:

```text
GitVersion:"v1.33.13"
```

Também:

```bash
kubectl version --client
```

Resultado:

```text
Client Version: v1.33.13
```

E nos workers:

```bash
kubelet --version
```

Resultado:

```text
Kubernetes v1.33.13
```

Portanto os componentes ficaram alinhados na versão:

```text
Kubernetes 1.33.13
```

---

# 18. Inicialização do Control Plane

Na `k8s-master` executamos:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.10 \
  --pod-network-cidr=10.244.0.0/16
```

Esse foi um dos comandos mais importantes do laboratório.

## --apiserver-advertise-address

```text
--apiserver-advertise-address=192.168.56.10
```

Informa ao kubeadm qual endereço o API Server deve anunciar para comunicação com os outros nós.

Utilizamos o IP Host-Only da master:

```text
192.168.56.10
```

## --pod-network-cidr

```text
--pod-network-cidr=10.244.0.0/16
```

Define a rede que será utilizada pelos Pods.

Escolhemos:

```text
10.244.0.0/16
```

Essa escolha está relacionada ao Flannel que instalamos posteriormente.

---

# 19. O que o kubeadm init criou

Durante o `kubeadm init`, foram criados certificados e configurações do cluster.

Entre os componentes configurados estavam:

* API Server;
* Controller Manager;
* Scheduler;
* etcd;
* kubelet;
* kubeconfig;
* CoreDNS;
* kube-proxy.

Também foi criado o arquivo:

```text
/etc/kubernetes/admin.conf
```

Esse arquivo permite que um administrador utilize o `kubectl` para acessar o cluster.

No final do processo apareceu:

```text
Your Kubernetes control-plane has initialized successfully!
```

Isso confirmou que o control plane foi criado com sucesso.

---

# 20. Configuração do kubectl

Para permitir que o usuário `vagrant` utilizasse o `kubectl`, executamos:

```bash
mkdir -p $HOME/.kube
```

Cria o diretório:

```text
~/.kube
```

Depois:

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

Copia a configuração administrativa para:

```text
~/.kube/config
```

Finalmente:

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Altera o proprietário do arquivo para o usuário atual.

Assim, podemos executar:

```bash
kubectl
```

sem precisar utilizar `sudo` toda hora.

---

# 21. Primeiro teste do cluster

Executamos:

```bash
kubectl get nodes
```

Inicialmente apareceu:

```text
k8s-master   NotReady
```

Isso não significava que o `kubeadm init` havia falhado.

O motivo era que ainda não tínhamos instalado uma rede de Pods.

---

# 22. Instalação do Flannel

Executamos:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

O Flannel é uma implementação de rede para Kubernetes.

Ele fornece conectividade entre os Pods distribuídos pelos diferentes nós.

O comando criou recursos como:

```text
namespace/kube-flannel
serviceaccount/flannel
clusterrole/flannel
clusterrolebinding/flannel
configmap/kube-flannel-cfg
daemonset.apps/kube-flannel-ds
```

---

# 23. Verificação do Flannel

Executamos:

```bash
kubectl get pods -n kube-flannel
```

Inicialmente o Pod estava:

```text
0/1 Init:1/2
```

Isso significava que ainda estava executando sua fase de inicialização.

Depois:

```bash
kubectl get pods -n kube-flannel
```

retornou:

```text
1/1 Running
```

Isso confirmou que o Flannel estava funcionando.

Então verificamos:

```bash
kubectl get nodes
```

A master passou para:

```text
k8s-master   Ready
```

---

# 24. Criação do comando para adicionar workers

No control plane executamos:

```bash
sudo kubeadm token create --print-join-command
```

O comando gerou um comando semelhante a:

```bash
kubeadm join 192.168.56.10:6443 --token ... \
--discovery-token-ca-cert-hash sha256:...
```

Esse comando contém as informações necessárias para que um novo nó seja autenticado e entre no cluster.

Ele possui:

## kubeadm join

Informa que estamos adicionando um nó existente a um cluster.

## 192.168.56.10:6443

É o endereço e porta do Kubernetes API Server.

A porta:

```text
6443
```

é a porta padrão do Kubernetes API Server.

## --token

É utilizado para autenticar o processo de ingresso do novo nó.

## --discovery-token-ca-cert-hash

Permite verificar a autoridade certificadora do cluster durante o processo de descoberta.

---

# 25. Entrada do Worker 1

No `k8s-worker1` executamos o comando `kubeadm join`.

O processo terminou com:

```text
This node has joined the cluster
```

Também foi informado:

```text
Certificate signing request was sent to apiserver and a response was received.
```

Isso significa que o worker conseguiu:

1. encontrar a API Server;
2. enviar sua solicitação;
3. obter aprovação;
4. configurar sua conexão segura;
5. iniciar o kubelet no cluster.

---

# 26. Entrada do Worker 2

O mesmo processo foi realizado no:

```text
k8s-worker2
```

O resultado também foi:

```text
This node has joined the cluster
```

Portanto os dois workers foram adicionados com sucesso.








