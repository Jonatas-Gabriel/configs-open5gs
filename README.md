Passo a passo para instalar o lxc, criar os containers já com o ubuntu instalado e verificar se os containers foram instalados:
Neste caso, cada container deve ser criado em máquinas virtuais diferentes a fim de validar os teste.

1- Instala o lxc:
sudo snap install lxd 

2- Verifica se o lxc está instalado corretamente:
lxc --version

3- Inicializa os dois containers com o ubuntu instalado:
lxc launch images:ubuntu/22.04 open5gs-core
lxc launch images:ubuntu/22.04 ueransim-ran
sudo lxd init

4- Lista oc containers criados:
lxc list

5- Entrar nos containers e instalar os ambientes do open5gs e ueransim:
5.1 - Instalar o open5gs no container do open5gs-core:
sudo lxc exec open5gs-core -- bash

apt update && apt install software-properties-common -y
add-apt-repository ppa:open5gs/latest -y
apt update
apt install open5gs -y

5.2 - Instalar o ueransim-ran no container ueransim-ran:
sudo lxc exec ueransim-ran -- bash

Instala as dependencias: 
apt install -y \
git \
make \
gcc \
g++ \
cmake \
libsctp-dev \
lksctp-tools \
iproute2 \
net-tools \
curl

clona o repositório:
git clone https://github.com/aligungr/UERANSIM.git

Entra no diretório do UERAMSIM:
cd UERANSIM

Dentro do repositório:
make

Verificar os arquivos contruídos ao final do comando make:
ls build/

Após isso, subir o ip 10.45.0.1 (ogstun) na interface ogstun e 10.108.79.151 na eth0 (eth0) par criar o tunel no container do opne5gs.
No Container do UERAMSIN, setar o ip 10.108.79.66 (eth0) na interface eth0 para dar inicio aos testes.

Testes no LXC:

No Core

systemctl restart open5gs-amfd open5gs-upfd (Reiniciar os serviços da amf e upf)

Passo 1: Ligar o Servidor de Destino (No container open5gs-core)
Aceda ao terminal do container do Core e inicie o receptor de tráfego do iPerf3:

1. Fatia eMBB (Throughput / Vazão)
Como o iPerf3 dá erro ao tentar alcançar a rede de dados tunelada externa (10.45.0.1), vamos realizar o teste de vazão apontando diretamente para o IP do container do Core na rede local do LXC (10.62.119.151). Isso mede a capacidade máxima de vazão de pacotes entre os containers através do switch virtual (lxdbr0).

Preparação (No Container open5gs-core)
Inicie o servidor iPerf3 em modo TCP:

iperf3 -s -p 5201

No Container ueransim-ran

iperf3 -c 10.62.119.151 -t 60 -P 4 - Dispara um teste simulando quatro conexões ao mesmo tempo para estressar e testar a vazão / Throughput.

2. Fatia URLLC (Latência e Jitter)
Para a latência e o jitter, o protocolo ICMP (ping) é o mais indicado em ambientes virtuais porque ele opera diretamente na camada de rede (Camada 3), ignorando as travas de descritores de sockets que afetam o iPerf3.

Teste de Latência RTT e Estabilidade (No Container ueransim-ran)
Como o celular virtual do UERANSIM está autenticado e conectado (CM-CONNECTED), a interface uesimtun0 está criada. Vamos forçar o envio de 100 pacotes em direção ao IP interno do gNB ou do Core para capturar a latência do túnel:

ping -I uesimtun0 -c 100 10.62.119.151

Passo 2: Ativar as Regras de Encaminhamento no Kernel (No container open5gs-core)
O UPF precisa de permissão no kernel do container para rotear pacotes da rede virtual 5G para o mundo externo:

Bash
sysctl -w net.ipv4.ip_forward=1

3. Jitter e Perda UDP (URLLC):

Bash
iperf3 -c 10.62.119.151 -u -b 50M -t 60




