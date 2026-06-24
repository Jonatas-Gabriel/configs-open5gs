# Laboratório 5G Core com Open5GS e UERANSIM

Projeto acadêmico focado na implementação e validação de uma rede 5G Core virtualizada utilizando containers **LXC**, **Open5GS** para o núcleo da rede (5GC) e **UERANSIM** para simulação da RAN (Radio Access Network).

---

## 📋 Arquitetura do Lab
Este laboratório simula um ambiente real de telecomunicações, onde a separação entre o núcleo (Core) e a RAN é mantida em containers isolados:
* **Container `open5gs-core`**: Responsável pela sinalização e processamento do core (AMF, UPF, SMF, etc).
* **Container `ueransim-ran`**: Simula a estação rádio base (gNB) e o Equipamento de Usuário (UE).

## 🚀 Guia de Implementação

### 1. Preparação do Ambiente (LXC/LXD)
Instale e inicialize o gerenciador de containers:
```bash
sudo snap install lxd
sudo lxd init
lxc --version

2. Provisionamento dos Containers
Crie os nós isolados para a topologia:

lxc launch images:ubuntu/22.04 open5gs-core
lxc launch images:ubuntu/22.04 ueransim-ran
lxc list

3. Configuração do Core (Open5GS)
Dentro do container open5gs-core:

sudo lxc exec open5gs-core -- bash
# Instalação dos pacotes
apt update && apt install software-properties-common -y
add-apt-repository ppa:open5gs/latest -y
apt update && apt install open5gs -y

4. Configuração da RAN (UERANSIM)
Dentro do container ueransim-ran:

sudo lxc exec ueransim-ran -- bash
# Dependências necessárias
apt install -y git make gcc g++ cmake libsctp-dev lksctp-tools iproute2 net-tools curl

# Compilação
git clone [https://github.com/aligungr/UERANSIM.git](https://github.com/aligungr/UERANSIM.git)
cd UERANSIM
make

🧪 Procedimentos de Validação e Testes
O sucesso da implementação é verificado através de testes de vazão (throughput) e latência.

1. Teste de Vazão (Throughput)
Utilizamos o iPerf3 para medir a capacidade do switch virtual (lxdbr0) entre os containers.

No open5gs-core (Servidor):

iperf3 -s -p 5201

No ueransim-ran (Cliente):

iperf3 -c <IP_DO_CORE> -t 60 -P 4

2. Teste de Latência (URLLC)
A latência é medida via ICMP diretamente no túnel de dados do UE (uesimtun0).

# Executar no container ueransim-ran
ping -I uesimtun0 -c 100 <IP_DESTINO>

🛠️ Tecnologias Utilizadas
Virtualização: LXC/LXD

Core 5G: Open5GS

Simulação RAN: UERANSIM

Redes: Protocolo SCTP, Túneis TUN/TAP

Documentação desenvolvida para fins acadêmicos - IFPB.

---

### 1. Preparação do Ambiente (Docker)

## ⚙️ Execução e Testes de Carga

Após a instalação, siga estes procedimentos para subir a infraestrutura e realizar as validações de desempenho.

### 1. Inicialização do Ambiente
Suba os serviços do 5G Core e a simulação de rádio (gNB e UE):

```bash
sudo docker compose -f sa-deploy.yaml up -d
sudo docker compose -f nr-gnb.yaml up -d
sudo docker compose -f nr-ue.yaml up -d

2. Configuração do Servidor de Testes
Para realizar as medições, subiremos um container Ubuntu dedicado para rodar o servidor iPerf3:

# Criar e iniciar o container servidor
sudo docker run -d --name servidor_testes_1 --network docker_open5gs_default ubuntu sleep infinity

# Iniciar o servidor iPerf3 em background
sudo docker exec -d servidor_testes_1 iperf3 -s

# Verificar o IP do servidor (necessário para os comandos de teste abaixo)
sudo docker exec -it servidor_testes_1 hostname -I

3. Validação de Desempenho (KPIs)
A. Teste eMBB (Vazão / Throughput)
Este teste força o envio contínuo de pacotes TCP durante 30 segundos para medir a capacidade máxima de banda que o UPF consegue processar:

sudo docker exec -it nr_ue iperf3 -c <IP_DO_SERVIDOR> -B 192.168.100.2 -t 30

B. Teste URLLC (Latência e Estabilidade)
Comunicações críticas exigem baixa latência e estabilidade (Jitter).

Medição de Latência (ICMP): Disparo de 50 pacotes para cálculo de RTT.

sudo docker exec -it nr_ue ping -c 50 -I uesimtun0 <IP_DO_SERVIDOR>

Análise: Observe os valores min/avg/max/mdev. O avg (média) representa a latência, e o mdev (desvio padrão) indica a instabilidade da conexão.

Medição de Jitter (UDP): Injeção de 10 Mbps constantes para identificar engasgos e perda de pacotes.

sudo docker exec -it nr_ue iperf3 -c <IP_DO_SERVIDOR> -B 192.168.100.2 -u -b 10M -t 30





