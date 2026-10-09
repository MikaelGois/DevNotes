---
title: Criando um cluster de computadores com Apache Hadoop
type: docs
weight: 1
editURL: "https://devnotes.msglabs.com.br/articles/hadoop-cluster/"
prev: /articles/2025/08/1-zabbix-and-grafana
---

Este artigo foi originalmente criado em 2023 como parte do projeto de avaliação da 2ª unidade da disciplina de Arquitetura de Computadores no curso de bacharelado em Ciência da Computação do Instituto Federal de Sergipe, curso do qual sou discente.  

Mantinha o documento apenas no meu Google Drive, porém achei que seria uma boa ideia escrever aqui e revisar parte do conteúdo.  

No trabalho da faculdade, tínhamos o seguinte cenário: 3 SBCs *Raspberry Pi*, onde uma era o ***Name Node*** (também chamada de ***main***/***master***), ou seja, controlava todo o *cluster* e duas atuaram como ***Data Nodes*** (também chamadas de ***nodes***/***slaves***) que eram responsáveis pelo processamento propriamente dito.
É possível incluir a máquina *main* como um dos *nodes*, ou seja, além de coordenar todo o *cluster*, ela também processa os dados, porém, não é essa abordagem adotada aqui, na verdade, essa revisão usará máquinas virtuais e uma configuração melhorada. No entanto, haverá uma indicação para aqueles que decidirem colocar a *main* para processar os dados juntamente com os *nodes*.

> [!IMPORTANT]
> Os valores abordados e os parâmetros usados foram utilizados conforme a necessidade do projeto e dos testes realizados, e portanto não devem ser seguidos ao pé da letra.  
> Ajuste todos os parâmetros conforme as necessidades do seu projeto e a capacidade do seu hardware.

## Atualizando os repositórios e pacotes:

Antes de iniciar, é importante atualizar os repositórios e pacotes instalados no sistema. No nosso caso, estamos usando um sistema baseado em Debian, então basta digitar o seguinte comando:

```bash
sudo apt update && sudo apt upgrade
```

## Passos necessários para a criação do cluster:

Os passos a seguir são fundamentais para o funcionamento do *cluster*:

### 1 - Instalação do Java openJDK (main/nodes):

Instale o Java openJDK tanto na máquina *main* quanto nas máquinas *node*:
```bash
sudo apt install openjdk-11-jdk
```




### 2 - Download do Hadoop (main/nodes):

Use o comando abaixo para realizar o *download* da versão 3.3.6 do Hadoop:
```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
```

Se houver problemas, acesse o repositório e baixe o pacote correspondente:  
[https://dlcdn.apache.org/hadoop/common/](https://dlcdn.apache.org/hadoop/common/)

> [!WARNING]
> O nome do pacote deve se parecer com `hadoop-VERSÃO.tar.gz`!

Descompacte o pacote:   
```bash
tar xzf hadoop-3.3.6.tar.gz
```

De preferência, renomeie a pasta apenas para “hadoop” e mova a pasta para um local que possa ser acessado facilmente, como `/` ou `/usr/local/`, o comando a seguir realiza as duas operações:
```bash
sudo mv hadoop-3.3.6 /usr/local/hadoop
```




### 3 - Configuração Hadoop (main/nodes):

Acesse o script de *environment* do hadoop:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

Procure por `export JAVA_HOME`, remova a indicação de comentário (`#`) e indique o caminho do Java OpenJDK:
```sh {filename="hadoop-env.sh"}
export JAVA_HOME=/usr/lib/jvm/java-1.11.0-openjdk-amd64
```

> [!WARNING]
> O nome do pacote deve corresponder com a versão da arquitetura do sistema!  
> Para saber o nome do arquivo você pode navegar até `/usr/lib/jvm` e dentro da pasta listar os diretórios com o comando `ls`.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.



### 4 - Configurando o Environment (main/nodes):

Acesse o arquivo de *environment*:
```bash
sudo nano /etc/environment
```

Indique o caminho de *PATH* do hadoop e o caminho do JAVA_HOME:
```sh {filename="environment"}
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/local/hadoop/bin:/usr/local/hadoop/sbin"

JAVA_HOME="/usr/lib/jvm/java-1.11.0-openjdk-amd64"
```

> [!WARNING]
> O trecho `/usr/local/hadoop/bin` e `/usr/local/hadoop/sbin` pode mudar dependendo de onde está o seu Hadoop, verifique e mude o caminho se necessário.  
> O nome do pacote do Java deve corresponder com a versão da arquitetura do sistema!

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.




### 5 - Usuário hadoop (main/nodes):

> [!WARNING]
> O passo a seguir é necessário apenas caso o seu usuário padrão não seja hadoop. Caso o usuário hadoop já exista, basta pular para o próximo passo.

Adicione usuário hadoop:
```bash
sudo adduser hadoop
```

> [!TIP]
> As informações de nome, número, etc, podem ser ignoradas clicando `ENTER`.

Conceda permissões de administrador ao usuário hadoop e atribua a propriedade da pasta hadoop:
```bash
sudo usermod -aG hadoop hadoop
sudo chown hadoop:root -R /usr/local/hadoop/
sudo chmod g+rwx -R /usr/local/hadoop/
sudo adduser hadoop sudo
```




### 6 - Configurações de rede (main/nodes):

#### 6.1 - Habilite o SSH:

Digite um dos comandos abaixo para habilitar o serviço de SSH, o comando `enable` ativa o serviço para iniciar automaticamente durante a inicialização do sistema, e o comando `start` inicia o serviço imediatamente, o `&&` permite que ambos os comandos sejam executados em sequência. O comando `enable --now` combina as duas ações, ativando o serviço para iniciar automaticamente e iniciando-o imediatamente sem a necessidade de rodar os comandos separadamente ou com o uso de `&&`.

```bash
sudo systemctl enable ssh && sudo systemctl start ssh
```

ou 

```bash
sudo systemctl enable --now ssh
```

#### 6.2 - Configure o IP estático (Debian):

> [!CAUTION]
> No nosso cenário, os testes estavam sendo realizados na faculdade, e para evitar maiores problemas, colocamos as máquinas em uma rede isolada conectadas apenas em um switch L2 simples.  
> Dependendo da configuração, a máquina poderá perder o acesso a internet!  
> Então, se tiver alguma configuração opcional que precise baixar pacotes da internet, como programas de monitoramento, pode ser uma boa hora para realizar essa configuração.

Acesse o arquivo de configuração de IP: 
```bash
sudo nano /etc/network/interfaces
```

No arquivo, você encontrará as configurações de interfaces de redes, a *interface* primária será algo como:           
```sh {filename="interfaces"}
# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet dhcp
```

Remova o parâmetro `dhcp` adicione as configurações de IP da máquina. Exemplo:  
```sh {filename="interfaces"}
allow-hotplug enp0s3
iface enp0s3 inet static
    address 192.168.0.X
    netmask 255.255.255.0
    gateway 192.168.0.1
    dns-nameservers 192.168.0.1 8.8.8.8
    dns-search cluster.local

# 'allow-hotplug enp0s3': pode ser 'auto enp0s3'.
# 'iface enp0s3 inet static:' Substitua pelo nome da interface e desabilite o DHCP.
# 'address 192.168.0.X': Define o endereço IP da máquina.
# 'netmask 255.255.255.0': Define a máscara de sub-rede.
# 'gateway 192.168.0.1': Define o endereço do Gateway.
# 'dns-nameservers 192.168.0.1 8.8.8.8': Define os endereços dos servidores DNS, ex.: 8.8.4.4 9.9.9.9 1.1.1.1
# 'dns-search cluster.local': Define os domínios de busca. Pode ser outro domínio, ex.: lab.local.
```

> [!NOTE]
> O `X` será o número da máquina. Por exemplo, `192.168.0.10/24` para a *main*/*master*.  
> No `dns-nameservers` você pode configurar mais de um servidor DNS, como o do google: `8.8.8.8` e `8.8.4.4`, ou o IP do seu roteador caso tenha um na rede.  
> Você pode configurar outras faixas de IP, porém as máquinas so irão conseguir se comunicar se estiverem na mesma rede.

> [!WARNING]
> Se a rede onde as máquinas estiverem conectadas possuir servidor DHCP ativo, atente-se para colocar endereços IPs fora da faixa do servidor DHCP. No cenário abordado aqui, as máquinas estão conectadas em um switch e estão em uma rede isolada.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

Aplique as novas configurações de rede:
```bash
sudo ifdown enp0s3 && sudo ifup enp0s3
```

Caso não funcione, você pode aplicar as configurações reiniciando o computador ou o serviço de *network*.
Caso esteja acessando a máquina via ssh, provavelmente irá perder a conexão e pode ter problemas para conectar com o novo IP.

Para reiniciar o serviço:
```bash
sudo systemctl restart networking
```

Reiniciar a máquina:
```bash
sudo reboot
```

#### 6.3 - Configure o IP estático (Ubuntu):

> [!CAUTION]
> No nosso cenário, os testes estavam sendo realizados na faculdade, e para evitar maiores problemas, colocamos as máquinas em uma rede isolada conectadas apenas em um switch L2 simples.  
> Dependendo da configuração, a máquina poderá perder o acesso a internet!  
> Então, se tiver alguma configuração opcional que precise baixar pacotes da internet, como programas de monitoramento, pode ser uma boa hora para realizar essa configuração.

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

Digite o endereço IP na *interface* desejada. Exemplo:
```yaml {filename="01-network.yaml"}
network:
    version: 2
    renderer: networkd
    ethernets:
        enp0s3:
            dhcp4: false
            addresses:
            - 192.168.0.X/24
            routes:
                - to: default
                  via: 192.168.0.1
            nameservers:
                search:
                    - cluster.local
                addresses:
                    - 192.168.0.1
                    - 8.8.8.8
# RESPEITE A INDENTAÇÃO!
# 'enp0s3': Substitua pelo nome da interface, pode ser enp0s3, eth0, etc.
# 'dhcp4: false' desabilita o DHCP.
# 'addresses: 192.168.0.X/24' Define o endereço IP da máquina.
# 'routes: to: default' Define a rota padrão.
# 'routes: via: 192.168.0.1' Define o endereço do Gateway.
# 'nameservers: search:' Define os domínios de busca. Pode ser outro domínio, ex.: lab.local.
# 'nameservers: addresses:' Define os endereços dos servidores DNS, ex.: 8.8.4.4 9.9.9.9 1.1.1.1
```

> [!NOTE]
> O `X` será o número da máquina. Por exemplo, `192.168.0.10/24` para a *main*/*master*.  
> Caso decida utilizar o arquivo `50-cloud-init.yaml` é necessário desativar o **cloud init** conforme instruido nos comentários do inicio do arquivo.  
> Você pode configurar outras faixas de IP, porém as máquinas so irão conseguir se comunicar se estiverem na mesma rede.

> [!WARNING]
> Se a rede onde as máquinas estiverem conectadas possuir servidor DHCP ativo, atente-se para colocar endereços IPs fora da faixa do servidor DHCP. No cenário abordado aqui, as máquinas estão conectadas em uma rede isolada.

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

#### 6.4 - Configurar o arquivo de host e nome de host:

Acesse o arquivo de *hosts*:
```bash
sudo nano /etc/hosts
```

No arquivo, você deverá inserir os IPs e o *hostname* das máquinas, por exemplo:
```sh {filename="hosts"}
192.168.0.10 main
192.168.0.11 node1
192.168.0.12 node2
```

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

Para alterar o nome do *host*, acesse:  
```bash
sudo nano /etc/hostname
```

O nome do *hostname* deve corresponder à máquina. Exemplo: "node1" para a máquina `node1`.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

Reinicie a máquina:
```bash
sudo reboot
```




### 7 - Configurar acesso SSH (main):

Caso o usuário `hadoop` não seja o usuário padrão e você tenha criado o usuário na etapa `5 - Usuário hadoop`, mude para o usuário `hadoop`:
```bash
su - hadoop
```

Execute o comando a seguir para gerar uma chave ssh:  
```bash
ssh-keygen -t rsa
```

> [!TIP]
> Quando for solicitado para preencher o local onde a *key* será criada e a *passphrase*, basta ignorar clicando `ENTER`.

Envie essa chave para as outras máquinas:
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop@X
```

> [!NOTE]
> O parâmetro “X” corresponde ao nome da máquina. Exemplo: `hadoop@node1` para enviar para a máquina `node1`.

> [!WARNING]
> É necessário enviar as chaves da *main* para todos os outros *nodes*.




### 8 - Configurações do Hadoop - Core, HDFS, MapReduce, YARN - (main):

#### 8.1 - Configurar o arquivo Core do hadoop:

Acesse o arquivo:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/core-site.xml
```

Entre as tags `<configuration>` e `</configuration>` insira as seguintes informações:
```xml {filename="core-site.xml"}
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://main:9000</value>
</property>
<property>
    <name>hadoop.http.staticuser.user</name>
    <value>hadoop</value>
</property>
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro                     | Função                                                                                                                    |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `fs.defaultFS`                | URI padrão do sistema de arquivos Hadoop.                                                                                 |
| `hadoop.http.staticuser.user` | Define qual usuário será assumido pelo servidor HTTP do Hadoop, o que permite gerenciar os arquivos via *interface* web.  |

{{% /details %}}

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.


#### 8.2 - Configurar o arquivo HDFS do Hadoop:

Acesse o arquivo:  
```bash
sudo nano /usr/local/hadoop/etc/hadoop/hdfs-site.xml
```

Entre as tags `<configuration>` e `</configuration>` insira as seguintes informações:  
```xml {filename="hdfs-site.xml"}
<property>
    <name>dfs.namenode.name.dir</name>
    <value>/usr/local/hadoop/data/nameNode</value>
</property>
<property>
    <name>dfs.datanode.data.dir</name>
    <value>/usr/local/hadoop/data/dataNode</value>
</property>
<property>
    <name>dfs.replication</name>
    <value>2</value>
</property>
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro               | Função                                                  |
| ----------------------- | ------------------------------------------------------- |
| `dfs.namenode.name.dir` | Diretório local para metadados do NameNode.             |
| `dfs.datanode.data.dir` | Diretório local onde os DataNodes armazenam os blocos.  |
| `dfs.replication`       | Número de réplicas por bloco no HDFS.                   |

{{% /details %}}

> [!NOTE]
> O valor padrão é 3, o que significa que cada bloco de dados será replicado em 3 *DataNodes* diferentes. Isso garante alta disponibilidade e tolerância a falhas, mas também aumenta o uso de espaço em disco.  
> A configuração depende da quantidade de *nodes* que você possui e da resiliência desejada para o *cluster*.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

#### 8.3 - Configurar o arquivo MapReduce do Hadoop:

Acesse o arquivo:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/mapred-site.xml
```

Entre as tags `<configuration>` e `</configuration>` insira as seguintes informações:
```xml {filename="mapred-site.xml"}
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
<property>
    <name>yarn.app.mapreduce.am.env</name>
    <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
</property>
<property>
    <name>mapreduce.map.env</name>
    <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
</property>
<property>
    <name>mapreduce.reduce.env</name>
    <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
</property>
<property>
    <name>mapreduce.application.classpath</name>
    <value>
        /usr/local/hadoop/share/hadoop/mapreduce/*,
        /usr/local/hadoop/share/hadoop/mapreduce/lib/*,
        /usr/local/hadoop/share/hadoop/common/*,
        /usr/local/hadoop/share/hadoop/common/lib/*,
        /usr/local/hadoop/share/hadoop/yarn/*,
        /usr/local/hadoop/share/hadoop/yarn/lib/*,
        /usr/local/hadoop/share/hadoop/hdfs/*,
        /usr/local/hadoop/share/hadoop/hdfs/lib/*
    </value>
</property>
<!-- OPCIONAL - Configurações para visualizar o historico dos trabalhos finalizados.
Complementar ao `yarn.log-aggregation-enable` -->
<property>
    <name>mapreduce.jobhistory.address</name>
    <value>main:10020</value>
</property>
<property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>main:19888</value>
</property>
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro                             | Função                                                                                                                                                        |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mapreduce.framework.name`            | Define qual *framework* de execução será utilizado pelo *MapReduce*. O valor `yarn` indica que o **YARN (Yet Another Resource Negotiator)** será utilizado.   |
| `yarn.app.mapreduce.am.env`           | Define variáveis de ambiente para o **ApplicationMaster** de *MapReduce*.                                                                                     |
| `mapreduce.map.env`                   | Define variáveis de ambiente para as tarefas *Map*.                                                                                                           |
| `mapreduce.reduce.env`                | Define variáveis de ambiente para as tarefas *Reduce*.                                                                                                        |
| `mapreduce.application.classpath`     | Define o classpath necessário para execução dos trabalhos *MapReduce*.                                                                                        |
| `mapreduce.jobhistory.address`        | Endereço e porta onde o **JobHistory Server** escuta para requisições RPC de clientes CLI/API.                                                                |
| `mapreduce.jobhistory.webapp.address` | Endereço e porta da *interface* **web (HTTP)** do *JobHistory Server*.                                                                                        |

{{% /details %}}

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.


#### 8.4 - Configurar o arquivo YARN do Hadoop:

Acesse o arquivo:  
```bash
sudo nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Entre as tags `<configuration>` e `</configuration>` insira as seguintes informações:
```xml {filename="yarn-site.xml"}
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>main</value>
</property>
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
<property>
    <name>yarn.nodemanager.auxservices.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
</property>
<!-- OPCIONAL - Configurações de log aggregation -->
<property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
</property>
<property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <value>172800</value>
</property>
<property>
  <name>yarn.log.server.url</name>
  <value>http://main:19888/jobhistory/logs</value>
</property>
<property>
    <name>yarn.nodemanager.remote-app-log-dir</name>
    <value>/tmp/logs</value>
</property>
<property>
    <name>yarn.nodemanager.remote-app-log-dir-suffix</name>
    <value>logs</value>
</property>
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro                                              | Função                                                                                       |
| ------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `yarn.resourcemanager.hostname`                        | Host onde o *ResourceManager* do YARN está escutando.                                        |
| `yarn.nodemanager.aux-services`                        | Serviço auxiliar habilitado no *NodeManager*.                                                |
| `yarn.nodemanager.auxservices.mapreduce.shuffle.class` | Classe Java que implementa o serviço de *shuffle*.                                           |
| `yarn.log-aggregation-enable`                          | Ativa a **agregação de logs** dos *containers* após o término das aplicações.                |
| `yarn.log-aggregation.retain-seconds`                  | Tempo em segundos que os logs agregados devem ser mantidos no HDFS.                          |
| `yarn.log.server.url`                                  | Configura a URL base para acessar os logs agregados das aplicações usando redirecionamento do log do *ResourceManager* para o *JobHistory Server*.                                       |
| `yarn.nodemanager.remote-app-log-dir`                  | Diretório HDFS onde os logs agregados das aplicações serão armazenados.                      |
| `yarn.nodemanager.remote-app-log-dir-suffix`           | Sufixo do caminho final de log remoto (útil para organizar subpastas por aplicação/usuário). |

{{% /details %}}

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

#### 8.5 - Configurar o arquivo de nodes no hadoop:

Acesse o arquivo:  
```bash
sudo nano /usr/local/hadoop/etc/hadoop/workers
```

Retire o nome *localhost* e adicione os *hostnames* dos *nodes*. Exemplo:
```sh {filename="workers"}
node1
node2
```

Se você quiser manter a máquina *main* como um *node*, adicione o parâmetro `main`:
```sh {filename="workers"}
main
node1
node2
```

Nesse guia, manteremos apenas as máquinas `node1` e `node2`.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

#### 8.6 - Enviar a pasta com os arquivos modificados para os nodes:

Execute o comando abaixo:
```bash
scp /usr/local/hadoop/etc/hadoop/* X:/usr/local/hadoop/etc/hadoop/
```

> [!NOTE]
> O parâmetro “X” corresponde ao nome da máquina. Exemplo: "node1:/usr/local/hadoop/etc/hadoop/" para enviar para a máquina `node1`.

> [!WARNING]
> É necessário enviar os arquivos modificados da *main* para todos os outros *nodes*.

#### 8.7 - Exportação dos paths:

Digite os comandos abaixo **em todas as máquinas** no usuário Hadoop para exportar os *PATH* das aplicações:
```bash
export HADOOP_HOME="/usr/local/hadoop"
export HADOOP_COMMON_HOME="/usr/local/hadoop"
export HADOOP_CONF_DIR="/usr/local/hadoop/etc/hadoop"
export HADOOP_HDFS_HOME="/usr/local/hadoop"
export HADOOP_MAPRED_HOME="/usr/local/hadoop"
export HADOOP_YARN_HOME="/usr/local/hadoop"
```




### 9 - Criar a pasta nameNode (main):

Digite o seguinte comando apenas na máquina *main*:
```bash
mkdir -p /usr/local/hadoop/data/nameNode
```




### 10 - Criar a pasta dataNode (nodes):

Digite o seguinte comando apenas nas máquinas *nodes*:  
```bash
mkdir -p /usr/local/hadoop/data/dataNode
```




### 11 - Formatar o Hadoop Distributed File System - HDFS (main):

Digite o comando abaixo para carregar as variáveis de ambiente:
```bash
source /etc/environment
```

Digite o comando abaixo para realizar a formatação do HDFS:
```bash
hdfs namenode -format
```

> [!WARNING]
> É comum realizar a exclusão das pastas nameNode e dataNode para a correção de alguns possíveis erros, ou após alterações nos arquivos do hadoop, ambos os casos realizar uma nova formatação do HDFS é obrigatório para efetivar as alterações!




### 12 - Inicializar, monitorar e finalizar o cluster (main):

#### 12.1 - Inicializando o cluster:

Sempre que você desejar inicializar o *cluster*, você terá que primeiro carregar as variáveis de ambiente primeiro:  
```bash
source /etc/environment
```

Em seguida, digite o seguinte comando para inicializar o HDFS:
```bash
start-dfs.sh
```

Logo após, digite o seguinte comando para inicializar o YARN:
```bash
start-yarn.sh
```

Para iniciar todos os serviços do Hadoop de uma só vez, você pode usar o comando:
```bash
start-all.sh
```

Caso tenha configurado o **JobHistory Server**, digite o seguinte comando para iniciar o serviço:
```bash
mapred --daemon start historyserver
```

> [!NOTE]
> Caso apareça um erro "main: hadoop@main: Permission denied (publickey,password).", isso indica que o serviço de SSH não está conseguindo autenticar a máquina *main* com a chave SSH. Para corrigir, envie a chave SSH da máquina *main* para ela mesma usando o comando `ssh-copy-id`: `ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop@main`

Para encerrar os serviços do *cluster*, basta substituir `start` por `stop` no comando.

Para verificar se o *cluster* foi inicializado corretamente, você pode digitar tanto na *main*, quanto nos *nodes*, o comando abaixo:  
```bash
jps
```


Esse comando deverá retornar algo semelhante às imagens abaixo:  
![Saída do JPS main][image1]
![Saída do JPS node 1][image2]
![Saída do JPS node 2][image3]

Se você estiver usando a máquina *main* como um *node*, também aparecerão os processos `DataNode` e `NodeManager`.

![Saída do JPS main como node][image4]

Caso esteja usando o **JobHistory Server** deverá aparecer na saída do `jps` o parâmetro `JobHistoryServer`. 


#### 12.2 - Monitorando o cluster:

##### 12.2.1 - Acessando a interface web do NameNode:

O comando `start-dfs`, além de inicializar o sistema de arquivos do Hadoop, também irá inicializar uma interface web com as informações sobre o *daemon* **NameNode**, nela você verá informações sobre o *cluster* e o *HDFS*.

Para acessar, digite no navegador: `ip_do_nameNode:9870` ou `main:9870`.

Na aba Datanodes, você verá os *nodes* que estão conectados ao *cluster*.

##### 12.2.2 - Acessando a interface web do ResourceManager:

O comando `start-yarn`, além de inicializar os serviços do *cluster*, também irá inicializar uma interface web com as informações sobre o *daemon* **ResourceManager**, e nela você poderá ver informações sobre os *nodes* conectados ao *cluster* e as aplicações submetidas, em execução e finalizadas.

Para acessar, digite no navegador: `ip_do_resourceManager:8088` ou `main:8088`.

Você deverá ver informações sobre o *cluster* semelhante ao exemplo anterior com informações sobre os *nodes* conectados, como: número de containers, memória, vCores, etc., além das informações sobre os trabalhos/aplicações como informado anteriormente.

##### 12.2.3 - Acessando o JobHistory Server:

O comando `mapred --daemon start historyserver` irá iniciar o *daemon* **MapReduce JobHistory Server**, que é responsável por armazenar o histórico dos trabalhos executados no *cluster*.

Com `ip_do_JobHistory:19888/jobhistory` ou `main:19888/jobhistory`, você poderá acessar o histórico dos trabalhos que foram executados no *cluster*.

#### 12.3 - Outros comandos úteis:

* `yarn node -list`: lista os *nodes* conectados ao *cluster*.
* `yarn application -list`: lista as aplicações em execução no *cluster*.
* `mapred --daemon stop historyserver`: para o serviço do  **JobHistory Server**.
* `hdfs help`: exibe os comandos disponíveis para o HDFS.
* `hdfs dfsadmin -safemode leave`: sai do modo de segurança do HDFS, permitindo que operações de escrita sejam realizadas (para o caso de aparecer um erro de escrita).
* `hdfs dfs -put /origem-local /destino-HDFS`: envia um arquivo do sistema de arquivos local para o HDFS.
* `hdfs dfs -get /origem-HDFS /destino-local`: baixa um arquivo do HDFS para o sistema de arquivos local.
* `hdfs dfs -ls /`: lista os arquivos e diretórios no HDFS.
* `hdfs dfs -mkdir /diretorio`: cria um diretório no HDFS.
* `hdfs dfs -rm /arquivo`: remove um arquivo do HDFS.
* `hdfs dfs -rmdir /diretorio`: remove um diretório vazio do HDFS.
* `hdfs dfs -rm -r /diretorio`: remove um diretório e todo o seu conteúdo do HDFS.




## Passos opcionais:

Os passos a seguir são opcionais, mas podem ser úteis para quem deseja monitorar o *cluster* ou realizar testes de benchmarks. Não é necessário implementar todas, fique à vontade para adicionar somente o que desejar.

### 13 - Configurando o **start-history** e **stop-history**:

Acesse o arquivo **bashrc**:
```bash
sudo nano ~/.bashrc
```

Cole o comando de inicialização e parada do **JobHistory Server**:
```sh {filename=".bashrc"}
# Iniciar o JobHistory Server
alias start-history='mapred --daemon start historyserver'

# Parar o JobHistory Server
alias stop-history='mapred --daemon stop historyserver'
```

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

Carregue o arquivo **bashrc**:
```bash
source ~/.bashrc
```

Agora você pode iniciar tudo com `start-all.sh && start-history` e parar tudo com `stop-all.sh && stop-history`.




### 14 - Configurando limites de recursos:

#### 14.1 - Limites para Aplicações nos nós (via YARN):

Quando não configurado, o YARN assume valores padrões definidos no `yarn-default.xml`, o que pode não ser o ideal em um *cluster* com nós  que possuem baixa capacidade de recursos, como o cluster de *Raspberry Pi*, por exemplo. O hadoop também é capaz de detectar automaticamente os recursos das máquinas. Para realizar essa configuração, veja a seção [15 - Configurando detecção automática de recursos](#15---configurando-detecção-automática-de-recursos).

<!-- Os valores padrão para o YARN são: -->
{{% details title="Valores padrão para o YARN (Clique para expandir)" closed="true" %}}

| Parâmetro                                     | Valor Padrão  | Explicação                                                                                                                                                                                                                            |
| --------------------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `yarn.nodemanager.resource.memory-mb`         | `-1`          | Se definido como `-1` e `yarn.nodemanager.resource.detect-hardware-capabilities` for `true`, será calculado automaticamente (no caso de Windows e Linux). Em outros casos, o padrão é 8192 MiB (8GiB).                                |
| `yarn.nodemanager.resource.cpu-vcores`        | `-1`          | Se definido como -1 e `yarn.nodemanager.resource.detect-hardware-capabilities` for `true`, o número será determinado automaticamente pelo hardware no caso de Windows e Linux. Em outros casos, o número de vCores é 8 por padrão.    |
| `yarn.scheduler.minimum-allocation-mb`        | `1024`        | Solicitações de memória inferiores a esse valor serão definidas com o valor desta propriedade. Além disso, um *node manager* configurado para ter menos memória do que esse valor será desligado pelo *resource manager*.             |
| `yarn.scheduler.maximum-allocation-mb`        | `8192`        | Solicitações de memória maiores que isso lançarão uma exceção `InvalidResourceRequestException`.                                                                                                                                      |
| `yarn.scheduler.minimum-allocation-vcores`    | `1`           | Solicitações inferiores a esse valor serão definidas com o valor desta propriedade. Além disso, um *node manager* configurado para ter menos núcleos virtuais do que esse valor será desligado pelo *resource manager*.               |
| `yarn.scheduler.maximum-allocation-vcores`    | `4`           | Solicitações maiores que isso lançarão uma exceção `InvalidResourceRequestException`.                                                                                                                                                 |

{{% /details %}}

>[!NOTE]
> Você pode consultar os valores padrão do YARN na [página oficial do yarn-default do Apache Hadoop](https://hadoop.apache.org/docs/r3.4.0/hadoop-yarn/hadoop-yarn-common/yarn-default.xml).  
> Para demais configurações, consulte a [documentação oficial do Apache Hadoop (3.4.0)](https://hadoop.apache.org/docs/r3.4.0/).

##### 14.1.1 - Configurando os limites de recursos para aplicações (nodes):

Acesse o arquivo `yarn-site.xml` em cada *node*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Entre as tags `<configuration>` e `</configuration>`, e abaixo das configurações anteriores, adicione as seguintes propriedades:
```xml {filename="yarn-site.xml"}
<!-- Limites para aplicações - nodes -->
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>3072</value>
</property>
<property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>3</value>
</property>
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro                                 | Função                                                                                    |
| ----------------------------------------- | ----------------------------------------------------------------------------------------- |
| `yarn.nodemanager.resource.memory-mb`     | Define a quantidade total de memória física, em MiB, que o YARN pode utilizar neste nó.   |
| `yarn.nodemanager.resource.cpu-vcores`    | Define o número de vCores (núcleos de CPU virtuais) que o YARN pode utilizar neste nó.    |

{{% /details %}}

> [!TIP]
> Defina a quantidade de memoria para 75-80% da RAM total da máquina. Se um nó tem 8GiB, use 6144 (6GiB).  
> Defina a quantidade de vCores para 75-80% do total de núcleos de CPU da máquina. Se um nó tem 6 cores, use 4 ou 5.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

##### 14.1.2 - Configurando os limites de recursos para Aplicações (main):

Acesse o arquivo `yarn-site.xml` na máquina *main*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Entre as tags `<configuration>` e `</configuration>`, e abaixo das configurações anteriores, adicione as seguintes propriedades:
```xml {filename="yarn-site.xml"}
<!-- Limites para aplicações - main -->
<property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>512</value>
</property>
<property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
    <value>2048</value>
</property>
<property>
    <name>yarn.scheduler.minimum-allocation-vcores</name>
    <value>1</value>
</property>
<property>
    <name>yarn.scheduler.maximum-allocation-vcores</name>
    <value>3</value>
</property>
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro                                     | Função                                                                                                                                                                                                                                                |
| --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `yarn.scheduler.minimum-allocation-mb`        | A menor unidade de memória que o YARN vai alocar para um contêiner. Geralmente, é bom alinhar isso com a RAM de um contêiner de *map/reduce*.                                                                                                         |
| `yarn.scheduler.maximum-allocation-mb`        | A maior quantidade de memória que **UMA ÚNICA** tarefa (contêiner) pode solicitar. Isso previne que uma tarefa mal configurada tente usar toda a memória de um *node*. Este valor **NÃO PODE** ser maior que `yarn.nodemanager.resource.memory-mb`.   |
| `yarn.scheduler.minimum-allocation-vcores`    | A menor unidade de vCores que o YARN vai alocar.                                                                                                                                                                                                      |
| `yarn.scheduler.maximum-allocation-vcores`    | O número máximo de vCores que **UMA ÚNICA** tarefa pode solicitar. Este valor **NÃO PODE** ser maior que `yarn.nodemanager.resource.cpu-vcores`.                                                                                                      |

{{% /details %}}

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

#### 14.2 - Limites para os Daemons do Hadoop (JVM Heap):

No caso dos *Daemons* do Hadoop, é importante configurar o tamanho do *heap* (tipo de estrutura de dados) da JVM para cada um deles. Isso garante que eles tenham memória suficiente para funcionar corretamente, especialmente em *clusters* maiores.
Não existe um valor padrão, pois ele escala de forma automática baseado na capacidade da máquina.

##### 14.2.1 - Configurando os limites de recursos para Daemons (main):

Acesse o arquivo `hadoop-env.sh` na máquina *main*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

No final do arquivo, adicione as seguintes linhas:
```sh {filename="hadoop-env.sh"}
# Heap para o NameNode (MUITO importante)
# Depende do número de arquivos/blocos no HDFS.
export HDFS_NAMENODE_OPTS="-Xms1024m -Xmx2048m"

# Heap para o ResourceManager
# Depende da quantidade de nós e apps.
export YARN_RESOURCEMANAGER_OPTS="-Xms1024m -Xmx2048m"

# Heap para o MapReduce JobHistory Server
# Aumente se a UI ficar lenta.
export HADOOP_JOB_HISTORYSERVER_OPTS="-Xms1024m -Xmx2048m"
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro                         | Função                                                                            |
| --------------------------------- | --------------------------------------------------------------------------------- |
| `HDFS_NAMENODE_OPTS`              | Define as opções de memória mínima e máxima para o *NameNode*.                    |
| `YARN_RESOURCEMANAGER_OPTS`       | Define as opções de memória mínima e máxima para o *ResourceManager*.             |
| `HADOOP_JOB_HISTORYSERVER_OPTS`   | Define as opções de memória mínima e máxima para o *MapReduce JobHistory Server*. |

{{% /details %}}

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

##### 14.2.2 - Configurando os limites de recursos para Daemons (nodes):

Acesse o arquivo `hadoop-env.sh` em cada *node*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

No final do arquivo, adicione as seguintes linhas:
```sh {filename="hadoop-env.sh"}
# Heap para o DataNode
# Geralmente não precisa de muita memória.
export HDFS_DATANODE_OPTS="-Xms512m -Xmx1024m"

# Heap para o NodeManager
# O próprio daemon não precisa de muito.
export YARN_NODEMANAGER_OPTS="-Xms512m -Xmx1024m"
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro                 | Função                                                            |
| ------------------------- | ----------------------------------------------------------------- |
| `HDFS_DATANODE_OPTS`      | Define as opções de memória mínima e máxima para o *DataNode*.    |
| `YARN_NODEMANAGER_OPTS`   | Define as opções de memória mínima e máxima para o *NodeManager*. |

{{% /details %}}

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

#### 14.3 - Limites para MapReduce:

Quando não configurado, o MapReduce assume valores padrões definidos no `mapred-default.xml`, o que pode não ser o ideal em um *cluster* com nodes que possuem baixa capacidade de recursos, como o cluster de *Raspberry Pi*, por exemplo.

<!-- Os valores padrão para o MapReduce são: -->
{{% details title="Valores padrão para o MapReduce (Clique para expandir)" closed="true" %}}

| Parâmetro                             | Valor Padrão  | Explicação                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mapreduce.map.memory.mb`             | `-1`          | A quantidade de memória a ser solicitada ao *scheduler* para cada tarefa de mapeamento. Se não for especificado ou não for positivo, será inferido de `mapreduce.map.java.opts` e `mapreduce.job.heap.memory-mb.ratio`. Se java-opts também não for especificado, o valor será definido como `1024`.                                          |
| `mapreduce.reduce.memory.mb`          | `-1`          | A quantidade de memória a ser solicitada ao *scheduler* para cada tarefa de redução. Se não for especificado ou não for positivo, será inferido de `mapreduce.reduce.java.opts` e `mapreduce.job.heap.memory-mb.ratio`. Se java-opts também não for especificado, definimos como `1024`.                                                      |
| `mapreduce.map.java.opts`             | `1024m`       | O valor efetivo é geralmente **-Xmx1024m**. (O Hadoop pode inferir este valor com base na memória do contêiner se não for definido explicitamente).                                                                                                                                                                                           |
| `mapreduce.reduce.java.opts`          | `1024m`       | O valor efetivo é geralmente **-Xmx1024m**.                                                                                                                                                                                                                                                                                                   |
| `mapreduce.job.heap.memory-mb.ratio`  | `0.8`         | A proporção entre o tamanho do *heap* e o tamanho do contêiner. Se `-Xmx` não for especificado, será calculado como (`mapreduce.{map\|reduce}.memory.mb` * `mapreduce.heap.memory-mb.ratio`). Se `-Xmx` for especificado, mas `mapreduce.{map\|reduce}.memory.mb` não, será calculado como (`heapSize` / `mapreduce.heap.memory-mb.ratio`).   |

{{% /details %}}

>[!NOTE]
> Você pode consultar os valores padrão do MapReduce na [página oficial do mapred-default do Apache Hadoop](https://hadoop.apache.org/docs/r3.4.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml).  
> Para demais configurações, consulte a [documentação oficial do Apache Hadoop (3.4.0)](https://hadoop.apache.org/docs/r3.4.0/).

Acesse o arquivo `mapred-site.xml` em cada *node*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/mapred-site.xml
```

Entre as tags `<configuration>` e `</configuration>`, e abaixo das configurações anteriores, adicione as seguintes propriedades:
```xml {filename="mapred-site.xml"}
<!-- Limites para Map e Reduce - main/nodes (recomendado) -->
<property>
    <name>mapreduce.map.memory.mb</name>
    <value>512</value>
</property>
<property>
    <name>mapreduce.map.java.opts</name>
    <value>-Xmx1024m</value>
</property>
<property>
    <name>mapreduce.reduce.memory.mb</name>
    <value>512</value>
</property>
<property>
    <name>mapreduce.reduce.java.opts</name>
    <value>-Xmx1024m</value>
</property>
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro                     | Função                                                                                                                                                                                                    |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mapreduce.map.memory.mb`     | Define o tamanho total, em MiB, do **contêiner YARN** que será solicitado para cada tarefa de **Map**.                                                                                                    |
| `mapreduce.map.java.opts`     | Define as opções da JVM, principalmente o heap máximo (`-Xmx`), para o processo Java que roda **dentro** do contêiner da tarefa de **Map**. Seu valor deve ser menor que `mapreduce.map.memory.mb`.       |
| `mapreduce.reduce.memory.mb`  | Define o tamanho total, em MiB, do **contêiner YARN** que será solicitado para cada tarefa de **Reduce**.                                                                                                 |
| `mapreduce.reduce.java.opts`  | Define as opções da JVM, principalmente o heap máximo (`-Xmx`), para o processo Java que roda **dentro** do contêiner da tarefa de **Reduce**. Seu valor deve ser menor que `mapreduce.reduce.memory.mb`. |

{{% /details %}}

> [!TIP]
> É uma **boa prática manter este arquivo de configuração sincronizado em todos os nós do cluster** (main e nodes) para garantir um comportamento consistente.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

#### 14.4 - Aplique as configurações:

Caso o cluster esteja rodando, é necessário reiniciar o *cluster* para que as alterações tenham efeito.

> [!NOTE]
> Não é necessário apagar as pastas `nameNode` e `dataNode` para aplicar as alterações de memória e formatar o HDFS novamente. Caso deseje, pode realizar, mas nesse caso não irei fazer isso.

Você pode fazer isso com os seguintes comandos:
```bash
stop-all.sh && start-all.sh
```

#### 14.5 - Considerações importantes:

Caso apareça um erro sobre o limite de recursos ao rodar um trabalho, é sinal de que os limites estão funcionando, mas é necessário entender como o YARN aloca realmente os recursos.

##### Cenário de exemplo:

- 1 - Você rodou um trabalho que pediu ao YARN para criar contêineres para as tarefas Map, e cada um desses contêineres precisava de 1536 MB de RAM.

- 2 - Você configurou o limite máximo teórico do YARN para 2048 MB (`yarn.scheduler.maximum-allocation-mb`).

- 3 - Porém, nos seus nós de trabalho, você configurou a memória total que cada nó oferece para o YARN como sendo apenas 1024 MB (`yarn.nodemanager.resource.memory-mb`).

- 4 - O YARN é inteligente. Ele não pode prometer um contêiner de 2048 MB se o seu nó mais forte só oferece 1024 MB no total. Portanto, ele reduz o limite máximo efetivo para o maior valor que seus nós realmente podem suportar, que no seu caso é 1024 MB.

- 5 - Seu trabalho, ao pedir 1536 MB, ficou exatamente nesse meio-termo: maior que a capacidade real dos nós (1024 MB), mas menor que a sua configuração teórica (2048 MB). E por isso a falha.

É importante entender que o YARN não vai alocar mais recursos do que o que os nós realmente podem oferecer. Portanto, é importante ter isso em mente na hora de definir os limites de recursos. Sempre alinhe as configurações de recursos entre o ***ResourceManager (main)*** e os ***NodeManagers (nodes)*** para evitar erros de alocação.




### 15 - Configurando detecção automática de recursos:

O Hadoop é capaz de detectar automaticamente algumas configurações de *hardware*, porém é necessário indicar que isso deve acontecer, caso contrário, se não houver essa configuração e não houver limites definidos, os valores padrão serão aplicados. Seguem instruções de como configurar a detecção automática.

#### 15.1 - Configure os nodes.

Acesse o arquivo `yarn-site.xml` em cada *node*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Entre as tags `<configuration>` e `</configuration>`, e abaixo das configurações anteriores, adicione as seguintes propriedades:
```xml {filename="yarn-site.xml"}
<!-- Detecção automática de recursos - nodes -->
<property>
    <name>yarn.nodemanager.resource.detect-hardware-capabilities</name>
    <value>true</value>
</property>
<property>
    <name>yarn.nodemanager.resource.system-reserved-memory-mb</name>
    <value>2048</value>
</property>
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro                                                 | Função                                                                                                                                                                                                        |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `yarn.nodemanager.resource.detect-hardware-capabilities`  | Quando `true`, o NodeManager ignora a configuração manual de **memória** e **vCores** e tenta detectar os valores reais da máquina.                                                                           |
| `yarn.nodemanager.resource.system-reserved-memory-mb`     | Quando a detecção está ligada, este valor é subtraído do total de RAM detectado. Isso garante que o Sistema Operacional e os *daemons* do Hadoop tenham memória para funcionar sem competir com os trabalhos. |

{{% /details %}}

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

#### 15.2 - Ajuste os limites da máquina main.

Caso não tenha definido os limites que uma aplicação pode utilizar, como mostrado na seção [14.1.2 - Configurando os limites de recursos para aplicações](#1412---configurando-os-limites-de-recursos-para-aplicações-main), vamos definir agora. 

Pode ser confuso pensar em definir limite se deveria ser algo automático, porém, o que definimos como automático é a detecção dos recursos da máquina, mas ainda é importante definir os limites mínimos e máximos que uma aplicação pode rodar nos *nodes* de forma que **mantenha a saúde** do nó e do *cluster*. A ideia é que o limite máximo não é mais um número arbitrário, mas sim um reflexo direto da capacidade real do seu hardware.

> [!NOTE]
> A configuração abordada aqui é mais simples do que a apresentada na seção [14.1.2 - Configurando os limites de recursos para aplicações](#1412---configurando-os-limites-de-recursos-para-aplicações-main), então recomendo fortemente que faça a leitura caso ainda não tenha feito.

Acesse o arquivo `yarn-site.xml` na *main*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Entre as tags `<configuration>` e `</configuration>`, e abaixo das configurações anteriores, adicione as seguintes propriedades:
```xml {filename="yarn-site.xml"}
<!-- Limites para aplicações - main -->
<property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>512</value> 
</property>
<property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
    <value>6144</value>
</property>
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro                                 | Função                                                                                                                                                                                                                                                        |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `yarn.scheduler.minimum-allocation-mb`    | A menor unidade de memória que o YARN vai alocar para um contêiner. Geralmente, é bom alinhar isso com a RAM de um contêiner de *map/reduce*.                                                                                                                 |
| `yarn.scheduler.maximum-allocation-mb`    | A maior quantidade de memória que **UMA ÚNICA** tarefa (contêiner) pode solicitar. Isso previne que uma tarefa mal configurada tente usar toda a memória de um *node*. Deve ser igual à memória disponível do seu maior nó (RAM Total - Memória Reservada).   |

{{% /details %}}

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

#### 15.3 - Aplique as configurações:

Caso o cluster esteja rodando, é necessário reiniciar o *cluster* para que as alterações tenham efeito.

> [!NOTE]
> Não é necessário apagar as pastas `nameNode` e `dataNode` para aplicar as alterações de memória e formatar o HDFS novamente. Caso deseje, pode realizar, mas não será o caso aqui.

Você pode fazer isso com os seguintes comandos:
```bash
stop-all.sh && start-all.sh
```




### 16 - Configurando o Swap:

O *Swap* é uma área no disco rígido que o sistema operacional usa como uma extensão da memória RAM. Ele é útil quando a RAM física está cheia, permitindo que o sistema continue funcionando, mas com uma queda significativa de desempenho. É importante configurar o *Swap* para evitar que o sistema fique sem memória e trave. Em máquinas com pouca RAM, isso pode ser especialmente importante.

#### 16.1 - Verifique o espaço de Swap:

Para verificar se o *Swap* está configurado, execute o seguinte comando:
```bash
sudo swapon --show
```

Se não houver saída, significa que o *Swap* não está configurado.

Se houver saída, você verá algo como:
```bash
NAME      TYPE SIZE USED PRIO
/swapfile file 2G   0B   -2
```

#### 16.2 - Crie o arquivo de Swap:

Para criar um arquivo de *Swap*, execute os seguintes comandos:
```bash
sudo fallocate -l 2G /swapfile
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro     | Função                                                                                        |
| ------------- | --------------------------------------------------------------------------------------------- |
| `-l 2G`       | Especifica o tamanho (*Length*) do arquivo. 2G para 2 Gigabytes. Você pode usar 4G, 8G, etc.  |
| `/swapfile`   | É o caminho onde o arquivo de *Swap* será criado.                                             |

{{% /details %}}

Caso o `fallocate` não esteja disponível, você pode usar o seguinte comando alternativo:
```bash
sudo dd if=/dev/zero of=/swapfile bs=1G count=2
```

{{% details title="Explicação dos parâmetros (Clique para expandir)" closed="true" %}}

| Parâmetro         | Função                                                |
| ----------------- | ----------------------------------------------------- |
| `if=/dev/zero`    | Especifica que o arquivo será preenchido com zeros.   |
| `of=/swapfile`    | É o caminho onde o arquivo de *Swap* será criado.     |
| `bs=1G`           | Especifica o tamanho do bloco como 1 Gibibyte.        |
| `count=2`         | Especifica que serão criados 2 blocos de 1 Gibibyte.  |

{{% /details %}}

##### 16.2.1 - Defina as permissões do arquivo de Swap:

Por segurança, apenas o usuário root deve ter permissão para ler e escrever no arquivo de swap.
```bash
sudo chmod 600 /swapfile
```

##### 16.2.2 - Formate o arquivo de Swap e ative-o:

Este comando prepara o arquivo para ser usado como swap:
```bash
sudo mkswap /swapfile
```

Agora, diga ao sistema para começar a usar este arquivo como memória swap:
```bash
sudo swapon /swapfile
```

#### 16.3 - Tornando o Swap permanente:

Para tornar o espaço de *Swap* permanente, adicione a seguinte linha ao arquivo `/etc/fstab`:
```bash
sudo nano /etc/fstab
```

Adicione a seguinte linha ao final do arquivo:
```bash
/swapfile none swap sw 0 0
```

#### 16.4 - Verificando novamente o espaço de Swap:

Após criar o espaço de *Swap*, execute novamente o comando:
```bash
sudo swapon --show
```

Você deverá ver a saída semelhante a:
```bash
NAME      TYPE SIZE USED PRIO
/swapfile file 2G   0B   -2
```

Também é possível verificar o uso do *Swap* com o comando:
```bash
free -h
```

A saída do `free -h` na linha "Swap" agora deve mostrar a capacidade total somada (o swap antigo, se houver, mais os 2 GB adicionados).




### 17 - Instalação do sistema de monitoramento do cluster (Zabbix e Grafana):

* [Instalando o Zabbix e Grafana para monitorar o cluster com Hadoop](/articles/2025/08/1-zabbix-and-grafana)




### 18 - Automatizando o processo de configuração do cluster com Ansible:

* [Automatizando a configuração de servidores com Ansible](/articles/2026/01/1-ansible)




### 19 - Testes de benchmark para medir o desempenho do cluster:

* [Realizando testes de benchmark com o Hadoop (em breve)](#)

<!-- Imagens -->

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,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>

[image4]: <data:image/png;base64,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>

