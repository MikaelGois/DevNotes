---
title: Automatizando a configuração de servidores com Ansible
type: docs
weight: 1
editURL: "https://devnotes.msglabs.com.br/articles/ansible/"
next: /articles/2025/08/2-dns-and-proxy-server-rockylinux
---

Gerir um cluster Hadoop manualmente é um excelente exercício didático, mas uma vez que a configuração desejada está definida, a automação da implementação de novos nós é essencial. O Ansible permite-nos transformar máquinas recém-instaladas em nós Hadoop funcionais com um único comando.

Este é o terceiro artigo de uma série de 4 artigos. Porém, as informações podem ser usadas para em qualquer outro cenário.

Além disso, a implementação do Ansible para automatizar o cluster ocorreu em um momento posterior ao descrito no artigo [Criando um cluster de computadores com Apache Hadoop](/articles/2025/07/1-hadoop-cluster), diferente de como descrito nesse artigo, onde a implementação do cluster inicialmente ocorreu como um trabalho da disciplina de Arquitetura de Computadores, o cenário no presente artigo ocorre em uma outra oportunidade de implementar o cluster novamente, onde as configurações utilizadas ja eram mais claras e maduras, o que facilitou a implementação do cluster de uma forma geral, e a implementação do Ansible para a automação.

Dito isso, a máquina usada para o Ansible também irá atuar como um roteador/gateway de internet para o cluster Hadoop, contornando o problema de acesso a internet do cluster que estava em uma rede local isolada devido as limitações da rede cabeada da faculdade. Isso foi possível porque a máquina possui adptador Wi-Fi, o que coloca o cluster sob as mesmas regras e limitações de acesso a internet que qualquer outro dispositivo usado por alunos.

Se a sua intenção é apenas implementar o Ansible, seja em um cluster ou em outro cenário, não precisa se preocupar com as configurações abordadas nesse artigo referente a outros componentes/funções, basta seguir as instruções para a instalação e configuração do Ansible, e a criação dos playbooks para a automação de tarefas conforme a sua necessidade.

Nesse artigo temos o seguinte cenário:
- Um cluster Hadoop composto por 3 máquinas, sendo uma delas o NameNode (main/master) e os outros dois DataNodes (nodes/slaves), um ja configurado e funcional, e outro que irá ser implementado com o Ansible.
- Um computador que atua como backoffice, onde o Zabbix, Graffana e o Ansible está instalado e a partir do qual monitoramos e gerenciamos o cluster. Essa máquina também atua como gateway de internet para o cluster, permitindo que os nós do cluster tenham acesso a internet através dela.

> [!NOTE]
> Apesar do cenário descrito acima, o presente artigo tem como foco a implementação do Ansible em si, juntamente das configurações necessárias para que ele possa cumprir sua função. As instruções para a instalação e configuração do Ansible, bem como a criação dos playbooks, podem ser adaptadas para outros cenários e necessidades. Não será abordada a configuração detalhada do cluster Hadoop, nem do Zabbix e Grafana, pois isso já foi abordado em outros artigos da série, e o foco aqui é a automação com Ansible.

> [!WARNING]
> É necessário instalar de forma prévia o sistema operacional em todos os dispositivos que serão afetados pela execução dos playbooks do Ansible. Além disso, é necessário ter configurado a rede e o acesso SSH. Este artigo não cobre a instalação do sistema operacional nem a configuração da rede e do acesso SSH.

É importante entender que o backoffice é apenas um ponto de gerenciamento e monitoramento para o cluster, ele não é um componente do cluster Hadoop, ou seja, ele não subistitui a máquina main que tem como função controlar o cluster Hadoop em sí. A máquina backoffice é dispensável para o funcionamento do cluster, o que ela nos permite é monitorar e gerenciar o cluster de forma mais eficiente e externa, sem adicionar carga a máquina main, que poderia acabar acumulando essas funções, mas sob o custo de sobrecarregar a máquina e afetar o desempenho do cluster.

Outros artigos da série:
* [Criando um cluster de computadores com Apache Hadoop](/articles/2025/07/1-hadoop-cluster)
* [Instalando o Zabbix e Grafana para monitorar o cluster com Hadoop](/articles/2025/08/1-zabbix-and-grafana)
* [Realizando testes de benchmark com o Hadoop (em breve)](#)

## O que é o Ansible e porquê usá-lo?

O Ansible é uma ferramenta de automação de TI que não precisa de agentes (agentless) instalados nos nós de destino. Ele funciona via SSH, o que o torna ideal para gerir dispositivos de forma remota. No contexto de clusters Hadoop, o Ansible permite-nos:
- Automatizar a instalação e configuração do Hadoop.
- Gerir atualizações e patches de segurança.
- Facilitar a escalabilidade do cluster, adicionando novos nós rapidamente.




## Configurando o Ambiente

### 1 - Instalação do Ansible (No Backoffice):

A instalação deve ser feita apenas na tua máquina de controlo (Backoffice/Master).

#### 1.1 - Atualizando os repositórios e pacotes:

Antes de iniciar, é importante atualizar os repositórios do sistema. No nosso caso, estamos usando um sistema baseado em Debian/Ubuntu, então basta digitar o seguinte comando:

```bash
sudo apt update
```

Depois instale o pacote `software-properties-common`, que é necessário para adicionar novos repositórios:

```bash
sudo apt install software-properties-common -y
```

#### 1.2 - Instalando o Ansible:

Primeiro, adicione o repositório oficial do Ansible:

```bash
sudo add-apt-repository --yes --update ppa:ansible/ansible
```

Agora, instale o Ansible e o `sshpass` (necessário para a autenticação baseada em senhas, que utilizaremos no primeiro acesso aos nós) com o seguinte comando:

```bash
sudo apt install ansible sshpass -y
```

Depois de finalizada a instalação, verifique a versão do Ansible para confirmar que tudo está correto:

```bash
ansible --version
```




### 2 - Estrutura do Projeto e Configuração Inicial:

#### 2.1 - Diretório do Projeto:

Dentro do diretório do Ansible em `/etc/ansible/`, você deverá criar uma estrutura organizada para armazenar seus playbooks, inventários e arquivos de configuração.

Exemplo de estrutura:
```bash
/etc/ansible/
├── files-to-send/           # Pasta com os arquivos a serem enviados
├── host_vars/               # Onde ficarão os passwords criptografados
├── playbooks/               # Os scripts de automação (YAML)
├── .vault_pass              # Password mestre para as credenciais
├── ansible.cfg              # Configurações de comportamento do Ansible
└── hosts.ini                # Inventário das máquinas geridas
```

Se a pasta `/etc/ansible/` não existir, você pode criar ela e as subpastas necessárias com o comando abaixo:

```bash
sudo mkdir -p /etc/ansible/{host_vars,files-to-send,playbooks}
```

Para criar os arquivos necessários, você pode usar o comando `touch`:

```bash
sudo touch /etc/ansible/{ansible.cfg,hosts.ini,.vault_pass}
```

#### 2.2 - Configurar o ansible.cfg:

Crie ou acesse este arquivo para evitar ter de confirmar a chave SSH de cada nó manualmente:

```bash
sudo nano /etc/ansible/ansible.cfg
```

Depois, adicione as configurações abaixo:

```ini {filename="ansible.cfg"}
[defaults]
inventory = hosts.ini
host_key_checking = False
vault_password_file = .vault_pass
remote_user = hadoop
private_key_file = ~/.ssh/ansible
```

#### 2.3 - Inventário de Hosts (hosts.ini):

O inventário define os nós do cluster Hadoop. Crie ou edite o arquivo `hosts.ini` com o seguinte comando:

```bash
sudo nano /etc/ansible/hosts.ini
```

Adicione o seguinte conteúdo:

```ini {filename="hosts.ini"}
[all:vars]
ansible_user=hadoop

# 1. Grupo para a máquina principal (Master/NameNode)
# Adicione a máquina principal do seu cluster aqui.
[main]
main ansible_host=192.168.0.100

# 2. Grupo para as máquinas de trabalho (Workers/DataNodes)
# Adicione todas as máquinas de trabalho aqui.
[nodes]
node1 ansible_host=192.168.0.101

# 3. Um grupo que contém todos os nós do Hadoop (main e nodes)
# Isso é útil para tarefas que precisam ser executadas em todo o cluster.
[cluster:children]
main
nodes

# 4. Grupo para os NOVOS nós a serem provisionados
# Para cada novo nó, defina seu IP e a variável 'novo_hostname'
[novos_nos]
node2 ansible_host=192.168.0.102 novo_hostname=node2
```

#### 2.4 - Gerando e Enviando Chaves SSH (Recomendado):

O acesso por chaves SSH é o método mais prático e seguro para gerenciar nós no Ansible. Caso opte por essa abordagem, primeiro gere um par de chaves no seu ambiente de backoffice especificando o nome do arquivo da chave como `ansible`:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible
```

> [!NOTE]
> Pressione `Enter` para todas as opções sugeridas pelo comando, evitando adicionar uma *passphrase* se desejar execuções totalmente automatizadas pelo Ansible.

Depois, distribua a chave pública para os nós do inventário atual (por exemplo, a `main` e o `node1`) que ainda necessitam de autenticação para permitir o acesso posterior sem senha de forma automatizada:

```bash
ssh-copy-id -i ~/.ssh/ansible.pub hadoop@192.168.0.100
```

Ao executar o comando acima a senha do usuário `hadoop` do nó de destino será requisitada pela primeira e única vez. Para os **novos** nós sem chave, iremos usar depois uma configuração especifica no playbook a fim de otimizar esse mesmo processo.

#### 2.5 - Configuração do Vault para Armazenar Credenciais:

O Ansible Vault permite criptografar dados sensíveis, como senhas de acesso aos nós e senhas de usuário, para que não fiquem expostos em texto plano no seu projeto. Primeiro iremos configurar a senha mestra para o Vault, e depois criaremos um script para facilitar a criação de arquivos de variáveis criptografados para cada nó do cluster.

Para evitar a necessidade de ter que criar manualmente o arquivo de variáveis criptografados para cada nó, o passo ["2.5.3 - Criando o script para gerar arquivos de variáveis criptografados em massa"](#253---criando-o-script-para-gerar-arquivos-de-variáveis-criptografados-em-massa) mostra como criar varios desses arquivos de uma só vez, o que é útil para cenários onde há muitos nós a serem provisionados.

A titulo de informação, no projeto do cluster Hadoop, optamos por usar o acesso por chave SSH porque o acesso por chave SSH é mais seguro e prático, e utilizar as variáveis criptografadas para armazenar as senhas do usuário `hadoop` criado em cada nó para ser utilizado para escalar privilégios em comandos com o uso do `sudo`. O problema de utilizar a chave SSH é que é preciso enviar a chave pública do backoffice para cada nó do cluster, então o primeiro acesso (de configuração) ainda exige o uso de senha, o que é resolvido adicionando também a senha para SSH no vault, e depois da configuração do nó, o acesso passa a ser feito por chave SSH.

##### 2.5.1 - Criando o arquivo de senha do mestre:

Primeiro, defina a senha mestra no arquivo `.vault_pass` que será responsável por criptografar e descriptografar os arquivos:

```bash
echo "sua_senha_super_secreta" | sudo tee /etc/ansible/.vault_pass
```

> [!WARNING]
> Mantenha este arquivo seguro e evite versioná-lo em sistemas como os do Git, pois quem tiver esta senha poderá acessar todas as credenciais criptografadas do projeto. Recomendamos restringir o acesso a ela apenas para o usuário que executará o Ansible com `sudo chmod 600 /etc/ansible/.vault_pass`.

##### 2.5.2 - Criando o script para gerar arquivos de variáveis criptografados para cada nó:

Para facilitar a criação dos arquivos de credenciais para cada host dentro da diretório `host_vars/`, vamos configurar o script `mk-host-vars.sh`:

```bash
sudo nano /etc/ansible/mk-host-vars.sh
```

No arquivo, iremos adicionar a senha do usuário `hadoop` criado em cada nó para ser utilizado tanto para o acesso SSH quanto para escalar privilégios em comandos com o uso do `sudo`, evitando a necessidade de digitar a senha para o acesso SSH e para o escalonamento de privilégios, por isso o uso das variáveis `ansible_ssh_pass` e `ansible_become_pass`, que são as variáveis usadas pelo Ansible para armazenar a senha de acesso SSH e a senha de escalonamento de privilégios, respectivamente.

Adicione o seguinte conteúdo no arquivo:

```bash {filename="mk-host-vars.sh"}
#!/bin/bash

# Solicita o nome do host e a senha
read -p "Digite o nome do usuário (ex: node2): " HOSTNAME
read -s -p "Digite a senha do usuário hadoop para criar o arquivo encriptado: " PASSWORD
echo ""

# Cria o arquivo de variáveis na pasta host_vars com a senha criptografada
echo "ansible_ssh_pass: $PASSWORD" > /tmp/$HOSTNAME.yml
echo "ansible_become_pass: $PASSWORD" >> /tmp/$HOSTNAME.yml

ansible-vault encrypt /tmp/$HOSTNAME.yml
sudo mv /tmp/$HOSTNAME.yml /etc/ansible/host_vars/$HOSTNAME.yml
sudo chown root:root /etc/ansible/host_vars/$HOSTNAME.yml

echo "Credenciais criptografadas salvas com sucesso em /etc/ansible/host_vars/$HOSTNAME.yml"
```

> [!NOTE]
> Não precisamos especificar o arquivo de senha do Vault no comando `ansible-vault encrypt` porque já definimos o caminho para o arquivo de senha mestre no `ansible.cfg` com a linha `vault_password_file = .vault_pass`, então o Ansible irá ler automaticamente a senha do arquivo `.vault_pass` para criptografar o arquivo de variáveis.

Dê permissões de execução para o script:

```bash
sudo chmod +x /etc/ansible/mk-host-vars.sh
```

Sempre que adicionar um novo nó ao seu inventário (`hosts.ini`), execute este script para armazenar as credenciais dele. Por exemplo, para o `node2`:

```bash
sudo /etc/ansible/mk-host-vars.sh
```

Ao rodar este script e preencher a senha solicitada do utilizador `hadoop` criado no `node2`, o arquivo `/etc/ansible/host_vars/node2.yml` criptografado será gerado. Quando o Ansible se conectar ao `node2`, ele usará este arquivo e lerá a senha automaticamente, viabilizando o provisionamento do novo nó de forma automatizada.

> [!NOTE]
> É necessário fazer isso para todas as máquinas do cluster, incluindo a máquina main, mesmo que ela já esteja configurada, para garantir que o Ansible tenha as credenciais necessárias para se conectar a todas as máquinas de forma automatizada.

##### 2.5.3 - Criando o script para gerar arquivos de variáveis criptografados em massa:

Nesse passo vamos criar um script para gerar as variáveis de hosts em masssa, o que é muito útil em cenários onde há muitas máquinas para serem provisionádas de uma vez.

Pimeiro, crie um arquivo chamado `hosts-pass.yaml`:

```bash
sudo nano /etc/ansible/hosts-pass.yaml
```

No arquivo adicionar os hosts e as senhas correspondentes a cada nó, seguindo o formato abaixo:

```yaml {filename="hosts-pass.yaml"}
main senha_main
node1 senha_node1
node2 senha_node2
node3 senha_node3
node4 senha_node4
node5 senha_node5
node6 senha_node6
node7 senha_node7
node8 senha_node8
node9 senha_node9
node10 senha_node10
```

Depois, crie o script `mk-host-vars-massa.sh`:

```bash
sudo nano /etc/ansible/mk-host-vars-massa.sh
```

Aqui, script irá criar o arquivo com as variáveis `ansible_ssh_pass` e `ansible_become_pass` para armazenar as senhas de acesso e escalonamento de privilégios, que são as senhas do usuário `hadoop` criado em cada máquina para ser utilizado para escalar privilégios em comandos com o uso do `sudo`, evitando a necessidade de digitar a senha em comandos que exigem privilégios elevados.

Adicione o seguinte conteúdo no arquivo:

```bash {filename="mk-host-vars-massa.sh"}
#!/bin/bash

# Arquivo de origem com os dados dos hosts
ARQUIVO_SENHAS="hosts-pass.yaml"

# Verifica se o arquivo de senhas existe
if [ ! -f "$ARQUIVO_SENHAS" ]; then
    echo "Erro: O arquivo '$ARQUIVO_SENHAS' não foi encontrado."
    exit 1
fi

# Cria o diretório host_vars se não existir
mkdir -p host_vars

# Lê o arquivo linha por linha
while read -r hostname password; do
    # Ignora linhas em branco ou comentadas
    [[ -z "$hostname" || "$hostname" =~ ^# ]] && continue

    echo "Processando host: $hostname"

    # Define o caminho do arquivo de saída
    output_file="host_vars/${hostname}.yml"

    # Cria o conteúdo YAML com a senha do host
    # e o criptografa usando ansible-vault, lendo do stdin.
    {
        printf -- 'ansible_ssh_pass: "%s"\n' "$password"
        printf -- 'ansible_become_pass: "%s"\n' "$password"
    } | ansible-vault encrypt --output "$output_file"

done < "$ARQUIVO_SENHAS"

echo "Processo concluído. Arquivos criados em 'host_vars/'."
```

> [!NOTE]
> Não precisamos especificar o arquivo de senha do Vault no comando `ansible-vault encrypt` porque já definimos o caminho para o arquivo de senha mestre no `ansible.cfg` com a linha `vault_password_file = .vault_pass`, então o Ansible irá ler automaticamente a senha do arquivo `.vault_pass` para criptografar o arquivo de variáveis.

Dê permissões de execução para o script:

```bash
sudo chmod +x /etc/ansible/mk-host-vars-massa.sh
```

Sempre que precisar rodar o script para criar os arquivos de variáveis criptografados para os hosts listados no `hosts-pass.yaml`, basta executar o seguinte comando:

```bash
sudo /etc/ansible/mk-host-vars-massa.sh
```




### 3 - Pré-configuração do Cluster Hadoop:

Para realizarmos a implementação do cluster Hadoop com o Ansible, é necessário que as máquinas estejam pré-configuradas com o sistema operacional, acesso SSH e rede configurados, e se for o caso, o usuário hadoop criado, além disso, precisamos de um padrão de configuração do Hadoop para ser replicado nos novos nós. 

Tendo isso em mente, configuramos a máquina main (NameNode) e um dos nodes (DataNodes) manualmente, seguindo as instruções do artigo [Criando um cluster de computadores com Apache Hadoop](/articles/2025/07/1-hadoop-cluster).

A máquina main tem configurações específicas para a sua função, enquanto as máquinas node tem configurações voltadas para o processamento de dados. A ideia é automatizar a implementação daquelas máquinas que de uma forma geral, seguem o mesmo padrão de configuração, ou seja, os DataNodes. Por isso também precisamos de um nó pré-configurado para servir como modelo para os novos nós a serem provisionados.

O que o Ansible irá fazer é pegar a configuração do node pré-configurado e replicar ela para os novos nós, garantindo que eles tenham as mesmas configurações e estejam prontos para se juntar ao cluster Hadoop.

Nada impede que o Ansible seja usado para configurar o todo o cluster, mas esse processo adicionaria uma complexidade maior a configuração do Ansible, havendo a necessidade de criar playbooks abrangendo a configuração de diferentes tipos de máquina e atendendo necessidades específicas de configuração para cada um dos tipos, o que seria uma configuração mais complexa e desnecessária para o nosso cenário, já que o foco é a automação da implementação dos DataNodes. Por isso, esse cenário específico, optamos por configurar a máquina main manualmente e um dos nodes para ter um ponto de referência claro para as configurações dos DataNodes.

O que precisamos para realizar a replicação dos nós é:
- Arquivo .pub da chave SSH da máquina main/master.
- Arquivo de variáveis de ambiente globais (`/etc/environment`), contendo o `PATH` do Hadoop e o `JAVA_HOME`.
- Arquivo .tar.gz do Hadoop, que será enviado para os novos nós e descompactado lá, evitando a necessidade de baixar o Hadoop em cada nó.
- Arquivos de configurações do Hadoop (tudo dentro da pasta `hadoop/etc/hadoop/`), que serão replicadas para os novos nós, garantindo que eles tenham as mesmas configurações e estejam prontos para se juntar ao cluster Hadoop.

Esses arquivos ficarão armazenados na pasta `files-to-send/` e serão referenciados nos playbooks do Ansible para serem enviados e configurados nos novos nós.

Você pode utilizar os comandos abaixo para copiar os arquivos do nó pré-configurado para a pasta `files-to-send/`:

```bash
sudo scp hadoop@192.168.0.100:/home/hadoop/.ssh/id_rsa.pub /etc/ansible/files-to-send/
sudo scp hadoop@192.168.0.100:/etc/environment /etc/ansible/files-to-send/
sudo scp hadoop@192.168.0.101:/home/hadoop/hadoop-3.3.6.tar.gz /etc/ansible/files-to-send/
sudo scp -r hadoop@192.168.0.101:/usr/local/hadoop/etc/hadoop/* /etc/ansible/files-to-send/hadoop-files
```

Caso haja outras configurações que você precise replicar para os novos nós, como por exemplo, arquivos de configuração do sistema, bashrc, scripts de inicialização, etc., você pode usar o comando `scp` para copiar esses arquivos para a pasta `files-to-send/` e depois referenciá-los nos playbooks do Ansible para serem enviados e configurados nos novos nós.




### 4 - Configurando o Backoffice como gateway e servidor de horas para o cluster Hadoop:

Como citado no artigo [Criando um cluster de computadores com Apache Hadoop](/articles/2025/07/1-hadoop-cluster), e no inicio desse artigo, o cluster foi implementando inicialmente em uma rede local isolada, sem acesso a internet, devido as limitações da rede cabeada da faculdade. Para contornar esse problema, o backoffice, que é a máquina onde o Ansible está instalado, foi configurado para atuar como um gateway de internet para o cluster Hadoop, permitindo que os nós do cluster tenham acesso a internet através dela.

#### 4.1 - Configurando IP estático:

Para configurar um IP estático na máquina backoffice (assumindo Ubuntu com Netplan), realize os passos abaixo:

Digite o seguinte comando para descobrir a *interface* de rede onde está configurado o IP:  
```bash
ip address
```

ou:
```bash
ifconfig
```

Navegue até a pasta `/etc/netplan`:
```bash
cd /etc/netplan
```

Agora liste os arquivos:
```bash
ls
```

Acesse ou crie o arquivo na pasta:
```bash
sudo nano 01-network.yaml
```

As configurações de rede devem seguir o formato abaixo, onde você deve substituir as informações de acordo com a sua rede e interface:
```yaml {filename="01-network.yaml"}
network:
    version: 2
    renderer: networkd
    ethernets:
        wlan0:
            dhcp4: true
        enp0s3:
            dhcp4: false
            addresses:
            - 192.168.0.1/24
            nameservers:
                search:
                    - lab.local
                addresses:
                    - 8.8.8.8
                    - 8.8.4.4
# RESPEITE A INDENTAÇÃO!
# 'wlan0': Substitua pelo nome da interface, pode ser enp0s3, eth0, etc.
# 'enp0s3': Substitua pelo nome da interface, pode ser enp0s3, eth0, etc.
# 'dhcp4: false' desabilita o DHCP.
# 'addresses: 192.168.0.X/24' Define o endereço IP da máquina. É comum usar o .1 para o gateway.
# 'nameservers: search:' Define os domínios de busca. Pode ser outro domínio, ex.: lab.local.
# 'nameservers: addresses:' Define os endereços dos servidores DNS, ex.: 8.8.4.4 9.9.9.9 1.1.1.1
```

> [!NOTE]
> Caso decida utilizar o arquivo `50-cloud-init.yaml` é necessário desativar o **cloud init** conforme instruido nos comentários do inicio do arquivo.  
> Você pode configurar outras faixas de IP, porém as máquinas so irão conseguir se comunicar se estiverem na mesma rede.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

Mude as permissões do arquivo para evitar problemas de acesso:
```bash
sudo chmod 600 01-network.yaml
```

Aplique as novas configurações de rede:
```bash
sudo netplan try
```

O comando acima primeiro testa a configuração e, caso as notações e as indentações estejam corretas, aparecerá uma opção para apertar `ENTER` confirmando as alterações dentro de 120 segundos. Caso o tempo se esgote e nenhuma ação seja tomada, as modificações são descartadas.

Caso deseje aplicar as modificações diretamente, sem testar, use o seguinte comando:
```bash
sudo netplan apply
```

#### 4.2 - Configurando o IP Forwarding:

Para permitir que o tráfego passe da rede interna para a rede Wi-Fi com internet, ative o IP Forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Para tornar essa alteração persistente após o reinício da máquina, edite o arquivo `/etc/sysctl.conf` e descomente ou adicione a seguinte linha:

```text
net.ipv4.ip_forward=1
```

#### 4.3 - Configurando o iptables para permitir o encaminhamento de pacotes:

Digite o comando abaixo para habilitar a regra de NAT `MASQUERADE` no `iptables`, substituindo `wlan0` pelo nome da sua interface Wi-Fi e `enp3s0` pela interface conectada à rede interna do cluster:
```bash
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
```

Digite o comando para permitir o roteamento entre a interface interna e a interface externa, substituindo `enp3s0` pela interface conectada à rede interna do cluster e `wlan0` pela interface com acesso à internet:
```bash
sudo iptables -A FORWARD -i enp3s0 -o wlan0 -j ACCEPT
```

Digite o comando para permitir o tráfego de retorno da internet para a rede interna, substituindo `wlan0` pela interface com acesso à internet e `enp3s0` pela interface conectada à rede interna do cluster:
```bash
sudo iptables -A FORWARD -i wlan0 -o enp3s0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Para salvar estas regras, utilize o pacote `iptables-persistent`:

```bash
sudo apt install iptables-persistent -y
```

Depois de instalar o `iptables-persistent`, salve as regras atuais:
```bash
sudo netfilter-persistent save
```

#### 4.4 - Configurando o Chrony para sincronização de tempo:

Para garantir que os nós do seu cluster Hadoop tenham a mesma sincronização temporal e evitar falhas no processamento por horários dessincronizados, usaremos o Backoffice como um servidor NTP local. Dessa forma, as máquinas em redes isoladas (ou se precisarem de tempo rápido sem requisições externas brutas) poderão se guiar pelo seu nó Ansible.

Para fazer isso de forma automatizada, vamos criar um script (`playbook`) exclusivo para sincronização, além de servir para regular clusters inteiros caso ocorra alguma dessincronia manual posterior:

```bash
sudo nano /etc/ansible/playbooks/sincronizar_tempo.yml
```

Insira o conteúdo deste playbook que atua em duas frentes: a Etapa 1 ativa o modo Servidor NTP no `localhost`, enquanto a Etapa 2 roda em `todas` as máquinas presentes no grupo `cluster` reconfigurando-os como Clientes do Ansible:

```yaml {filename="sincronizar_tempo.yml"}
# -----------------------------------------------------------------------------
# ETAPA 1: Configurar a Máquina Ansible (localhost) como Servidor NTP
# -----------------------------------------------------------------------------
- name: Configurar a máquina Ansible como servidor NTP
  hosts: localhost
  connection: local
  become: true
  vars:
    cluster_network: "192.168.0.0/24" # Substitua pela faixa de IP da sua rede do cluster
    chrony_conf_path: "{{ '/etc/chrony.conf' if ansible_os_family == 'RedHat' else '/etc/chrony/chrony.conf' }}"
    chrony_service_name: "{{ 'chronyd' if ansible_os_family == 'RedHat' else 'chrony' }}"

  tasks:
    - name: Definir o fuso horário para America/Sao_Paulo (UTC-3)
      timezone:
        name: America/Sao_Paulo

    - name: Instalar o pacote chrony no servidor
      ansible.builtin.package:
        name: chrony
        state: present

    - name: Permitir que a rede do cluster acesse o servidor NTP
      ansible.builtin.lineinfile:
        path: "{{ chrony_conf_path }}"
        line: "allow {{ cluster_network }}"
        state: present

    - name: Configurar o servidor para servir tempo mesmo se não sincronizado
      ansible.builtin.lineinfile:
        path: "{{ chrony_conf_path }}"
        line: "local stratum 10"
        state: present
      notify: Reiniciar o serviço chrony no servidor

    - name: Garantir que o serviço chrony esteja ativo e habilitado no servidor
      ansible.builtin.service:
        name: "{{ chrony_service_name }}"
        state: started
        enabled: yes

  handlers:
    - name: Reiniciar o serviço chrony no servidor
      ansible.builtin.service:
        name: "{{ chrony_service_name }}"
        state: restarted

# -----------------------------------------------------------------------------
# ETAPA 2: Configurar os Nós do Cluster como Clientes NTP
# -----------------------------------------------------------------------------
- name: Configurar os nós do cluster como clientes NTP
  hosts: cluster
  become: true
  vars:
    ntp_server_ip: "{{ hostvars['localhost']['ansible_default_ipv4']['address'] }}"
    chrony_conf_path: "{{ '/etc/chrony.conf' if ansible_os_family == 'RedHat' else '/etc/chrony/chrony.conf' }}"
    chrony_service_name: "{{ 'chronyd' if ansible_os_family == 'RedHat' else 'chrony' }}"

  tasks:
    - name: Definir o fuso horário para America/Sao_Paulo (UTC-3)
      timezone:
        name: America/Sao_Paulo

    - name: Instalar o pacote chrony no cliente
      ansible.builtin.package:
        name: chrony
        state: present

    - name: Remover configurações de NTP padrão (pool e server)
      ansible.builtin.lineinfile:
        path: "{{ chrony_conf_path }}"
        state: absent
        regexp: '^(pool|server)\s'

    - name: Adicionar o servidor NTP (máquina Ansible)
      ansible.builtin.lineinfile:
        path: "{{ chrony_conf_path }}"
        line: "server {{ ntp_server_ip }} iburst"
        state: present
      notify: Reiniciar o serviço chrony no cliente

    - name: Garantir que o serviço chrony esteja ativo e habilitado no cliente
      ansible.builtin.service:
        name: "{{ chrony_service_name }}"
        state: started
        enabled: yes

  handlers:
    - name: Reiniciar o serviço chrony no cliente
      ansible.builtin.service:
        name: "{{ chrony_service_name }}"
        state: restarted
```

Rode esse playbook pela primeira vez para estabelecer a sincronia com os hardwares já presentes (como o `main` e `node1`):

```bash
sudo ansible-playbook -i hosts.ini playbooks/sincronizar_tempo.yml --ask-pass
```




## Gerindo o Cluster com Ansible

Agora que o ambiente está preparado e já temos os arquivos base do Hadoop copiados para o diretório `files-to-send`, vamos criar o playbook principal. Criaremos um arquivo chamado `provisionar_cluster.yml` que vai automatizar todas as etapas.

Para facilitar o entendimento, vamos dividir as tarefas em dois blocos: **[1 - Configurações Essenciais](#1---configurações-essenciais)** e **[3 - Configurações Adicionais](#3---configurações-adicionais)**.

Você pode criar o arquivo do playbook com o seguinte comando:

```bash
sudo nano /etc/ansible/playbooks/provisionar_cluster.yml
```

### 1 - Configurações Essenciais:

As etapas a seguir configuram os nós na rede, instalam o Java e o Hadoop, e os acoplam ao cluster de fato.

#### 1.1 - Bootstrap da Chave SSH:

Adicione a **Etapa 1** no final do playbook para copiar a chave SSH pública do mestre (main) e do backoffice (ansible) para permitir comunicação posterior sem uso de senha:

```yaml {filename="provisionar_cluster.yml"}
# -----------------------------------------------------------------------------
# ETAPA 1: Bootstrap da Chave SSH (Usa Senha temporariamente)
# -----------------------------------------------------------------------------
- name: ETAPA 1 - Bootstrap da Chave SSH
  hosts: novos_nos # Limitado pelo --limit novos_nos
  become: false
  gather_facts: false
  tasks:
    - name: Copiar as chaves SSH públicas (Backoffice e Master) para o novo nó
      ansible.posix.authorized_key:
        user: "{{ ansible_user }}"
        state: present
        key: "{{ item }}"
      loop:
        - "{{ lookup('file', '/home/backoffice/.ssh/ansible.pub') }}"
        - "{{ lookup('file', '/etc/ansible/files-to-send/id_rsa.pub') }}"
```

#### 1.2 - Cabeçalho da Etapa 2 e Inicialização:

Iniciamos a **Etapa 2**, definimos as variáveis e resolvemos dependências iniciais e preparamos os Hosts:

```yaml {filename="provisionar_cluster.yml"}
# -----------------------------------------------------------------------------
# ETAPA 2: Provisionamento Completo do Nó (Usa Chave SSH)
# -----------------------------------------------------------------------------
- name: ETAPA 2 - Configurar o Novo Nó
  hosts: novos_nos
  become: true
  serial: 1
  vars:
    master_ip: "192.168.0.100" # ip da máquinas main/master
    zabbix_server_ip: "192.168.0.1" # ip da máquina do zabbix, apenas se for usar
    hadoop_user: "hadoop" # usuário criado em cada nó para rodar o Hadoop
    hadoop_archive: "hadoop-3.3.6.tar.gz" # nome do arquivo .tar.gz do Hadoop a ser enviado para os nós
    hadoop_extracted_dir: "hadoop-3.3.6"
    hadoop_install_dir: "/usr/local/hadoop" # diretório onde o Hadoop será instalado em cada nó
    chrony_conf_path: "{{ '/etc/chrony/chrony.conf' if ansible_os_family == 'Debian' else '/etc/chrony.conf' }}" # caminho do arquivo de configuração do chrony, apenas se for usar
    chrony_service_name: "{{ 'chrony' if ansible_os_family == 'Debian' else 'chronyd' }}" # nome do serviço do chrony, apenas se for usar
  tasks:
    - name: Coletar fatos da máquina Ansible (localhost)
      ansible.builtin.setup:
      delegate_to: localhost
      delegate_facts: true

    - name: Definir o hostname da máquina
      ansible.builtin.hostname:
        name: "{{ novo_hostname }}"
```

#### 1.3 - Sincronização de Data, Instalação do Java e Serviços Base:

O cluster necessita de sincronia temporal e do JDK. Insira imediatamente após o bloco acima (respeitando o recuo lógico da chave `tasks:` no seu YAML):

```yaml {filename="provisionar_cluster.yml"}
    - name: Configurar e sincronizar NTP
      block:
        - name: Definir o fuso horário para America/Sao_Paulo (UTC-3)
          timezone:
            name: America/Sao_Paulo

        - name: Instalar chrony
          ansible.builtin.package:
            name: chrony
            state: present

        - name: Limpar configurações NTP padrão
          ansible.builtin.lineinfile:
            path: "{{ chrony_conf_path }}"
            state: absent
            regexp: '^(pool|server)\s'

        - name: Adicionar servidor NTP (máquina Ansible)
          ansible.builtin.lineinfile:
            path: "{{ chrony_conf_path }}"
            line: "server {{ hostvars['localhost']['ansible_default_ipv4']['address'] }} iburst"
            state: present

        - name: Forçar reinicialização e sincronização do chrony
          ansible.builtin.service:
            name: "{{ chrony_service_name }}"
            state: restarted

        - name: Esperar o chrony sincronizar o tempo
          ansible.builtin.command: chronyc waitsync 30 0.5
          changed_when: false

    - name: Instalar OpenJDK 11
      block:
        - name: Tentar atualizar o cache do apt (ignora erros)
          ansible.builtin.apt:
            update_cache: yes
          ignore_errors: true
        - name: Instalar o pacote OpenJDK 11
          ansible.builtin.apt:
            name: openjdk-11-jdk
            state: present
            update_cache: no

    - name: Garantir que o serviço SSH esteja ativo e habilitado
      ansible.builtin.service:
        name: ssh
        state: started
        enabled: yes
```

#### 1.4 - Instalação e Configuração do Hadoop:

No mesmo nível da variável `tasks:`, inserimos os blocos de manipulação dos arquivos .tar.gz e perfis de inicialização do nó:

```yaml {filename="provisionar_cluster.yml"}
    - name: Instalação e Configuração do Hadoop
      block:
        - name: Verificar se o arquivo do Hadoop já existe no nó
          ansible.builtin.stat:
            path: "/home/{{ hadoop_user }}/{{ hadoop_archive }}"
          register: hadoop_archive_check

        - name: Copiar o arquivo do Hadoop para o nó (se não existir)
          ansible.builtin.copy:
            src: "/etc/ansible/files-to-send/{{ hadoop_archive }}"
            dest: "/home/{{ hadoop_user }}/{{ hadoop_archive }}"
            owner: "{{ hadoop_user }}"
            group: "{{ hadoop_user }}"
          when: not hadoop_archive_check.stat.exists

        - name: Extrair o arquivo do Hadoop (se /hadoop não existir)
          ansible.builtin.unarchive:
            src: "/home/{{ hadoop_user }}/{{ hadoop_archive }}"
            dest: "/home/{{ hadoop_user }}/"
            owner: "{{ hadoop_user }}"
            group: "{{ hadoop_user }}"
            remote_src: yes
            creates: "{{ hadoop_install_dir }}"

        - name: Mover e renomear o diretório do Hadoop para /hadoop
          ansible.builtin.command: "mv /home/{{ hadoop_user }}/{{ hadoop_extracted_dir }} {{ hadoop_install_dir }}"
          args:
            creates: "{{ hadoop_install_dir }}"

        - name: Copiar arquivos de configuração do Hadoop para o nó
          ansible.builtin.copy:
            src: "/etc/ansible/files-to-send/hadoop-files/"
            dest: "{{ hadoop_install_dir }}/etc/hadoop/"
            owner: "{{ hadoop_user }}"
            group: "{{ hadoop_user }}"

        # - name: Copiar o arquivo .bashrc customizado # Caso queira usar um .bashrc customizado para o usuário hadoop, com as variáveis de ambiente do Hadoop, etc.
        #   ansible.builtin.copy:
        #     src: "/etc/ansible/files-to-send/.bashrc"
        #     dest: "/home/{{ hadoop_user }}/.bashrc"
        #     owner: "{{ hadoop_user }}"
        #     group: "{{ hadoop_user }}"

        - name: Copiar variáveis de ambiente globais (/etc/environment)
          ansible.builtin.copy:
            src: "/etc/ansible/files-to-send/environment"
            dest: "/etc/environment"
            owner: root
            group: root
            mode: '0644'

        - name: Adicionar variáveis de ambiente do Hadoop ao .bashrc
          ansible.builtin.blockinfile:
            path: "/home/{{ hadoop_user }}/.bashrc"
            block: |
              # Variáveis de Ambiente do Hadoop
              export HADOOP_HOME="{{ hadoop_install_dir }}"
              export HADOOP_COMMON_HOME="{{ hadoop_install_dir }}"
              export HADOOP_CONF_DIR="{{ hadoop_install_dir }}/etc/hadoop"
              export HADOOP_HDFS_HOME="{{ hadoop_install_dir }}"
              export HADOOP_MAPRED_HOME="{{ hadoop_install_dir }}"
              export HADOOP_YARN_HOME="{{ hadoop_install_dir }}"
            marker: "# {mark} ANSIBLE MANAGED BLOCK HADOOP ENV"

        - name: Criar o diretório Data/HDFS para o DataNode
          ansible.builtin.file:
            path: "{{ hadoop_install_dir }}/data/datanode" # Pode ser outro caminho, dependendo da configuração do hdfs-site.xml
            state: directory
            owner: "{{ hadoop_user }}"
            group: "{{ hadoop_user }}"
```

#### 1.5 - Atualização do aruqivo de hosts e workers:

Após concluirmos a injeção do Hadoop nos nós novatos, avisamos o mestre (na **Etapa 3**, um novo nível no playbook) para que seu serviço de orquestração saiba do nó recém adicionado:

```yaml {filename="provisionar_cluster.yml"}
# -----------------------------------------------------------------------------
# ETAPA 3: Atualizar workers e hosts global
# -----------------------------------------------------------------------------
- name: ETAPA 3 - Atualizar workers e hosts em todas as as máquinas
  hosts: all
  become: true
  vars:
    master_ip: "192.168.0.100" # ip da máquinas main/master
  tasks:
    - name: Garantir que o bloco de hosts do cluster exista em /etc/hosts
      ansible.builtin.blockinfile:
        path: /etc/hosts
        block: |
          # Bloco de hosts do Cluster Hadoop
          {{ master_ip }} main # Use main ou master dependendo do nome da máquina
          {% for i in range(1, 21) %} # Aumente o range se tiver mais de 20 máquinas no cluster
          192.168.0.{{ 100 + i }} node{{ i }} # Mude a soma para a faixa de ip usada. Use node ou slave dependendo do nome da máquina
          {% endfor %}
        marker: "# {mark} ANSIBLE MANAGED BLOCK HADOOP"

    - name: Adicionar os novos nós ao arquivo de workers
      ansible.builtin.lineinfile:
        path: "/usr/local/hadoop/etc/hadoop/workers" # Use o caminho correto do arquivo de workers na main/master
        line: "{{ hostvars[item]['novo_hostname'] }}"
        state: present
      loop: "{{ ansible_play_batch }}"
      when: hostvars[item]['novo_hostname'] is defined
```


### 2 - Executando o Playbook:

Com todas as configurações e arquivos no lugar, você pode iniciar o provisionamento. 

Como configuramos o vault para armazenar as senha de SSH com `ansible_ssh_pass`, não será necessário o parâmetro que solicita a senha no terminal `--ask-pass` para a realizar a primeira tarefa do playbook, que é a cópia da chave SSH pública para os nós. Todas as tarefas subsequentes que exigirem autenticação SSH irá usar a chave SSH enviada para máquinas, e o escalonamento de privilégios com `sudo` usarão as variáveis `ansible_become_pass` dispensando também a necessidade de solicitar senha para o `sudo` durante a execução do playbook.

Assim, o comando para executar o playbook fica muito mais simples. Dentro do diretório `/etc/ansible`, basta rodar: 

```bash
sudo ansible-playbook -i hosts.ini playbooks/provisionar_cluster.yml
```

Um exemplo de como seria o comando sem o uso da senha SSH armazenada no vault e sem a indicação do caminho do vault no `ansible.cfg`, para comparação:

```bash
sudo ansible-playbook -i hosts.ini playbooks/provisionar_cluster.yml --vault-password-file .vault_pass --ask-pass
```

Com isso ja dá para perceber algumas coisas:
- Comandos que exigem senha, em um cenário onde o vault **exite** mas não está indicado previamente, precisam do parâmetro `--vault-password-file` seguido do caminho do arquivo de senha do vault.
- Comandos que exigem senha, em um cenário onde o vault **não está configurado**, precisam do parâmetro `--ask-pass` para solicitar a senha de SSH no terminal

Dependendo das credenciais armazenadas no vault, pode ocorrer a necessidade de fornecer a senha para algum passo do playbook que você estiver usando. Por exemplo, se o `ansible_become_pass` estiver armazenado no vault, mas o `ansible_ssh_pass` não estiver, o Ansible irá solicitar a senha para acesso SSH durante a execução da primeira tarefa do playbook que abordamos anteriormente, que é o envio da chave SSH pública para a máquina que está sendo configurada.

Você também pode executar playbooks especificando o grupo de hosts ou máquinas específicas usando o parâmetro `--limit`. Por exemplo, para executar o playbook apenas nos nós do cluster, você pode usar:

```bash
sudo ansible-playbook -i hosts.ini playbooks/provisionar_cluster.yml --limit cluster
```

Nesse caso, como esse playbook em especial indica o grupo em cada bloco, não é necessário usar o `--limit` para limitar a execução, mas imagine um cenário onde você tenha um playbook para executar comando genéricos (que irá ser apresentado aqui em [4.1 - Executando comandos genéricos em múltiplas máquinas](#41---executando-comandos-genéricos-em-múltiplas-máquinas)) nas máquinas do cluster, e queira rodar apenas em um subconjunto específico de máquinas, como nas máquinas do grupo `nodes`, para realizar alguma configuração ou teste específico, nesse caso o `--limit` seria muito útil para limitar a execução apenas para os hosts do grupo `nodes`:

```bash
sudo ansible-playbook -i hosts.ini playbooks/executar_comando.yml --limit nodes
```

Também é possível iniciar a partir de uma etapa específica do playbook usando o parâmetro `--start-at-task` seguido do nome da tarefa onde você deseja iniciar a execução. Por exemplo, se o nó que está sendo configurado ja possui algumas configurações prévias, seja por uma execução anterior que falhou em algum ponto, ou por alguma configuração manual, e você deseja iniciar a execução a partir da tarefa "Garantir que o bloco de hosts do cluster exista em /etc/hosts", você pode usar:

```bash
sudo ansible-playbook -i hosts.ini playbooks/provisionar_cluster.yml --start-at-task "Garantir que o bloco de hosts do cluster exista em /etc/hosts"
```

O playbook irá iniciar a execução a partir da tarefa especificada, ignorando as tarefas anteriores. Isso é útil para retomar a execução de um playbook a partir de um ponto específico, sem precisar executar novamente as tarefas que já foram concluídas com sucesso.

É possivel também, executar tarefas específicas dentro de um playbook usando o parâmetro `--tags` para marcar as tarefas que você deseja executar. Por exemplo, se você tiver uma tarefa marcada com a tag `instalar_vim`, você pode executar apenas essa tarefa usando:

```bash
sudo ansible-playbook -i hosts.ini playbooks/seu_playbook.yml --tags instalar_vim
```

Talves você ja tenha percebido que o uso desses parâmetros de controle de execução, como `--limit`, `--start-at-task` e `--tags`, são muito úteis para ter um controle mais granular sobre a execução dos playbooks, e que podem ser utilizados de forma combinada, permitindo que você execute apenas as partes necessárias, retome a execução a partir de um ponto específico ou execute a tarefa para um grupo específico de hosts, ou ainda uma combinação de tudo isso, o que pode economizar tempo e recursos, especialmente em playbooks mais longos ou complexos.




### 3 - Configurações Adicionais:

Podemos incluir otimizações de recursos ou integração de monitoramento. Retorne para as definições de `tasks:` da Etapa 2 ou adicione outras ao final se seguir blocos independentes.

#### 3.1 - Configuração de Swap (Opcional - Etapa 2)
Evita perda de alocação em ambientes pobres em RAM de DataNodes (adicione antes do fechamento da Etapa 2, caso queira embutir num único escopo de hosts):

```yaml {filename="provisionar_cluster.yml"}
    - name: Configurar swap de 1G
      block:
        - name: Criar arquivo de swap
          ansible.builtin.command: fallocate -l 1G /swapfile
          args:
            creates: /swapfile
        - name: Definir permissões do swapfile
          ansible.builtin.file:
            path: /swapfile
            mode: '0600'
        - name: Verificar se o swapfile já está ativo
          ansible.builtin.command: swapon --show
          register: active_swaps
          changed_when: false
        - name: Formatar o swapfile (apenas se não estiver ativo)
          ansible.builtin.command: mkswap /swapfile
          when: "'/swapfile' not in active_swaps.stdout"
        - name: Ativar o swapfile (apenas se não estiver ativo)
          ansible.builtin.command: swapon /swapfile
          when: "'/swapfile' not in active_swaps.stdout"
        - name: Adicionar swap ao /etc/fstab
          ansible.builtin.lineinfile:
            path: /etc/fstab
            line: '/swapfile none swap sw 0 0'
            regexp: '^/swapfile'
        - name: Configurar swappiness
          ansible.posix.sysctl:
            name: vm.swappiness
            value: '100'
            state: present
```

#### 3.2 - Integração com Zabbix (Monitoramento - Etapas 2 e 4)
Este fluxo se divide entre a instalação e inicialização do Daemon Zabbix no Nó Novo (dentro da Etapa 2) e o Cadastro via API Zabbix hospedada fora, executando no Localhost.

Inserindo os binários na máquina escrava:

```yaml {filename="provisionar_cluster.yml"}
    - name: Configurar Zabbix Agent (Modo Ativo)
      block:
        - name: Instalar zabbix-agent
          ansible.builtin.package:
            name: zabbix-agent
            state: present
        - name: Configurar ServerActive e Hostname no agente
          ansible.builtin.lineinfile:
            path: /etc/zabbix/zabbix_agentd.conf
            regexp: "^{{ item.key }}="
            line: "{{ item.key }}={{ item.value }}"
          loop:
            - { key: 'ServerActive', value: '{{ zabbix_server_ip }}' }
            - { key: 'Hostname', value: '{{ novo_hostname }}' }
          notify: Reiniciar Zabbix Agent

    - name: Reiniciar a máquina para aplicar todas as alterações
      ansible.builtin.reboot:
        msg: "Reiniciando o nó para aplicar a configuração de hostname e outras."
        post_reboot_delay: 30

  handlers:
    - name: Reiniciar Zabbix Agent
      ansible.builtin.service:
        name: zabbix-agent
        state: restarted
```

Na **Etapa 4** inserimos o cadastro interativo do endpoint usando uma Task exclusiva para manipulação remota REST (Anexe no final do seu `.yml`):

```yaml {filename="provisionar_cluster.yml"}
# -----------------------------------------------------------------------------
# ETAPA 4: Adicionar Host ao Zabbix via API
# -----------------------------------------------------------------------------
- name: ETAPA 4 - Adicionar Host ao Zabbix
  hosts: localhost # O Zabbix foi instalado na máquina Ansible, então rodamos localmente
  connection: local
  gather_facts: false
  vars:
    zabbix_server_url: "http://192.168.0.1/zabbix/api_jsonrpc.php"
    zabbix_api_user: "ansible" # Substitua pelo usuário do Zabbix API
  vars_files:
    - /etc/ansible/zabbix_secrets.yml # Arquivo vault com a senha do Zabbix API, você pode criar em host_vars
  tasks:
    - name: 1. Autenticar na API do Zabbix e obter token
      ansible.builtin.uri:
        url: "{{ zabbix_server_url }}"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: "user.login"
          params:
            user: "{{ zabbix_api_user }}"
            password: "{{ zabbix_api_password }}"
          id: 1
      register: zabbix_auth
      check_mode: no

    - name: 2. Obter ID do grupo 'Cluster' # Certifique-se de que o grupo exista no Zabbix antes de rodar
      ansible.builtin.uri:
        url: "{{ zabbix_server_url }}"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: "hostgroup.get"
          params:
            output: "extend"
            filter:
              name:
                - "Cluster" # Substitua pelo nome do grupo onde deseja adicionar os hosts
          auth: "{{ zabbix_auth.json.result }}"
          id: 2
      register: zabbix_group
      check_mode: no

    - name: Verificar se o grupo 'Cluster' foi encontrado
      ansible.builtin.assert:
        that:
          - zabbix_group.json.result | length > 0
        fail_msg: "ERRO: O grupo 'Cluster' não foi encontrado no Zabbix. Verifique o nome."

    - name: 3. Obter ID do template 'Linux by Zabbix agent active' # Certifique-se de que o template exista no Zabbix antes de rodar
      ansible.builtin.uri:
        url: "{{ zabbix_server_url }}"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: "template.get"
          params:
            output: "extend"
            filter:
              host:
                - "Linux by Zabbix agent active" # Substitua pelo nome do template que deseja obter
          auth: "{{ zabbix_auth.json.result }}"
          id: 3
      register: zabbix_template
      check_mode: no

    - name: Verificar se o template 'Linux by Zabbix agent active' foi encontrado
      ansible.builtin.assert:
        that:
          - zabbix_template.json.result | length > 0
        fail_msg: "ERRO: O template 'Linux by Zabbix agent active' não foi encontrado. Verifique o nome."

    - name: 4. Criar cada novo nó no Zabbix
      ansible.builtin.uri:
        url: "{{ zabbix_server_url }}"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: "host.create"
          params:
            host: "{{ hostvars[item]['novo_hostname'] }}"
            name: "{{ hostvars[item]['novo_hostname'] }}"
            inventory_mode: 0 # 0 = automatic
            groups:
              - groupid: "{{ zabbix_group.json.result[0].groupid }}"
            templates:
              - templateid: "{{ zabbix_template.json.result[0].templateid }}"
            # A seção 'interfaces' é removida para agentes ativos
          auth: "{{ zabbix_auth.json.result }}"
          id: 4
      loop: "{{ groups['novos_nos'] }}"
      when: hostvars[item]['novo_hostname'] is defined
      register: zabbix_host_creation
      check_mode: no
```

Com este guia o Ansible fica encarregado de injetar configurações críticas do cluster em novos hardwares bare-metal e deixá-los atrelados ao registro de visualização sob o Zabbix, permitindo o provisionamento simultâneo de inumeros nós.




### 4 - Outros playbooks uteis:

#### 4.1 - Executando comandos genéricos em múltiplas máquinas:

sudo ansible all -m shell -a "sudo shutdown now" --become