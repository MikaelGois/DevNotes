---
title: Criando um cluster de computadores com Apache Hadoop
type: docs
weight: 2
editURL: "https://devnotes.msglabs.site/articles/hadoop-cluster/"
---

<style>body {text-align: justify}</style>

Este artigo foi originalmente criado em 2023 como parte do projeto de avaliação da 2ª unidade da disciplina de Arquitetura de Computadores no curso de bacharelado em Ciência da Computação do Instituto Federal de Sergipe, curso do qual sou discente.  

Mantinha o documento apenas no meu Google Drive, porém achei que seria uma boa ideia escrever aqui e revisar parte do conteúdo.  

No trabalho da faculdade, tínhamos o seguinte cenário: 3 SBCs *Raspberry Pi*, onde uma era o ***Name Node*** (também chamada de ***main***/***master***), ou seja, controlava todo o *cluster* e duas atuaram como ***Data Nodes*** (também chamadas de ***nodes***/***slaves***) que eram responsáveis pelo processamento propriamente dito.
É possível incluir a máquina *main* como um dos *nodes*, ou seja, além de coordenar todo o *cluster*, ela também processa os dados, porém, não é essa abordagem adotada aqui, na verdade, essa revisão usará máquinas virtuais e uma configuração melhorada. No entanto, haverá uma indicação para aqueles que decidirem colocar a *main* para processar os dados juntamente com os *nodes*.

> [!IMPORTANT]
> Os valores abordados e os parâmetros usados foram utilizados conforme a necessidade do projeto e dos testes realizados, e portanto não devem ser seguidos a pé da letra.  
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

De preferência, renomeie a pasta apenas para “hadoop” e mova a pasta para `/usr/local/`, o comando a seguir realiza as duas operações:
```bash
sudo mv hadoop-3.3.6 /usr/local/hadoop
```




### 3 - Configuração Hadoop (main/nodes):

Acesse o script de *environment* do hadoop:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

Procure por `export JAVA_HOME`, remova o comentário e indique o caminho do java openJDK:
```sh {filename="hadoop-env.sh"}
export JAVA_HOME=/usr/lib/jvm/java-1.11.0-openjdk-amd64
```

> [!WARNING]
> O nome do pacote deve corresponder com a versão da arquitetura do sistema!  
> Para saber o nome do arquivo você pode navegar até `/usr/lib/jvm`.

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
> O passo a seguir é necessário apenas caso o seu usuário padrão não seja hadoop.

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

```bash
sudo systemctl enable ssh && sudo systemctl start ssh
```

#### 6.2 - Configure o IP estático (raspbian e debian):

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

# 'allow-hotplug enp0s3' pode ser 'auto enp0s3'
# 'iface enp0s3 inet static' substitua pelo nome da interface e desabilite o DHCP
# 'address 192.168.0.X' substitua pelo IP da máquina
# 'netmask 255.255.255.0' substitua pela máscara de sub-rede
# 'gateway 192.168.0.1' coloque o IP do seu gateway
# 'dns-nameservers 192.168.0.1 8.8.8.8' defina os endereços dos servidores DNS, ex.: 8.8.4.4 9.9.9.9 1.1.1.1
```

> [!NOTE]
> O `X` será o número da máquina. Por exemplo, `192.168.0.10/24` para a *main*/*master*.  
> No `dns-nameservers`você pode configurar mais de um servidor dns, como o do google: `8.8.8.8` e `1.1.1.1`, ou o IP do seu roteador caso tenha um na rede.  
> Você pode configurar outras faixas de IP, porém as máquinas so irão conseguir se comunicar se estiverem na mesma rede.

> [!WARNING]
> Se a rede onde as máquinas estiverem conectadas possuir servidor DHCP ativo, atente-se para colocar endereços IPs fora da faixa do servidor DHCP. No cenário abordado aqui, as máquinas estão conectadas em um switch e estão em uma rede isolada.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

Para configurar o serviço de resolução de nomes, acesse:  
```bash
sudo nano /etc/resolv.conf
```

Insira as informações:
```sh {filename="resolv.conf"}
domain home.local
search home.local
nameserver 192.168.0.1
nameserver 8.8.8.8
# 'domain home.local' ou outro domínio, ex.: cluster.local.
# 'search home.local' ou outros servidores dns, ex.: cluster.local.
# 'nameserver 192.168.0.1' ou outros servidores dns, ex.: 8.8.4.4 9.9.9.9 1.1.1.1 Um por linha!
```

Você pode aplicar as configurações reiniciando o computador ou o serviço de *network*.
Caso esteja acessando a máquina via ssh, provavelmente irá perder a conexão e pode ter problemas para conectar com o novo IP.

No meu caso, irei reiniciar a máquina:
```bash
sudo reboot
```

Para reiniciar o serviço:
```bash
sudo systemctl restart networking
```

#### 6.3 - Configure o IP estático (ubuntu):

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
                addresses:
                    - 192.168.0.1
                    - 8.8.8.8
# RESPEITE A INDENTAÇÃO!
# 'enp0s3' substitua pelo nome da interface, pode ser enp0s3, eth0, etc.
# 'dhcp4: false' desabilita o DHCP.
# 'addresses: 192.168.0.X/24' substitua pelo IP da máquina.
# 'gateway4: 192.168.0.1' coloque o IP do seu gateway.
# 'nameservers: addresses:' defina os endereços dos servidores DNS, ex.: 8.8.4.4 9.9.9.9 1.1.1.1
```

> [!NOTE]
> O `X` será o número da máquina. Por exemplo, `192.168.0.10/24` para a *main*/*master*.  
> Caso decida utilizar o arquivo `50-cloud-init.yaml` é necessário desativar o **cloud init** conforme instruido nos comentários do inicio do arquivo.  
> Você pode configurar outras faixas de IP, porém as máquinas so irão conseguir se comunicar se estiverem na mesma rede.

> [!WARNING]
> Se a rede onde as máquinas estiverem conectadas possuir servidor DHCP ativo, atente-se para colocar endereços IPs fora da faixa do servidor DHCP. No cenário abordado aqui, as máquinas estão conectadas em uma rede isolada.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

Aplique as novas configurações de rede:
```bash
sudo netplan try
```

O comando acima primeiro testa a configuração e, caso as notações e as indentações estejam corretas, aparecerá uma opção para apertar `ENTER` confirmando as alterações dentro de 120 segundos. Caso o tempo se esgote e nenhuma ação seja tomada, as modificações são descartadas.

Caso deseje aplicar as modificações diretamente, sem testar, use o seguinte comando:
```bash
sudo netplan apply
```

Para configurar o serviço de resolução de nomes, acesse:  
```bash
sudo nano /etc/resolv.conf
```

Insira as informações:
```sh {filename="resolv.conf"}
domain home.local
search home.local
nameserver 192.168.0.1
nameserver 8.8.8.8
# 'domain home.local' ou outro domínio, ex.: cluster.local.
# 'search home.local' ou outros servidores dns, ex.: cluster.local.
# 'nameserver 192.168.0.1' ou outros servidores dns, ex.: 8.8.4.4 9.9.9.9 1.1.1.1 Um por linha!
```

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

Você pode aplicar as configurações reiniciando o serviço do `resolv.conf`:
```bash
sudo systemctl restart resolvconf
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

O nome do *hostname* deve corresponder com a máquina. Exemplo: "node1" para a máquina `node1`.

Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair.

Reinicie a máquina:
```bash
sudo reboot
```




### 7 - Configurar acesso SSH (main):

Acesse o usuário hadoop:  
```bash
su - hadoop
```

Execute o comando a seguir para gerar uma chave ssh:  
```bash
ssh-keygen -t rsa
```

Quando for solicitado para preencher o local onde a *key* será criada e a *passphrase*, basta ignorar clicando `ENTER`.

Envie essa chave para as outras máquinas:
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop@X
```

> [!NOTE]
> O parâmetro “X” corresponde ao nome da máquina. Exemplo: `hadoop@node1` para enviar para a máquina `node1`.

> [!NOTE]
> É necessário enviar as chaves da *main* para todos os outros *nodes*, e dos *nodes* para os outros *nodes* e para a *main*.  
> Na duvida, envie para as três máquinas, assim não tem chance de erro.




### 8 - Configurações do Hadoop - core, hdfs, mapreduce, yarn - (main):

#### 8.1 - Configurar o arquivo core do hadoop:

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


#### 8.2 - Configurar o arquivo hdfs do Hadoop:

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

#### 8.3 - Configurar o arquivo mapreduce do Hadoop:

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


#### 8.4 - Configurar o arquivo yarn do Hadoop:

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
| `yarn.nodemanager.auxservices.mapreduce.shuffle.class` | Classe Java que implementa o serviço de shuffle.                                             |
| `yarn.log-aggregation-enable`                          | Ativa a **agregação de logs** dos *containers* após o término das aplicações.                |
| `yarn.log-aggregation.retain-seconds`                  | Tempo em segundos que os logs agregados devem ser mantidos no HDFS.                          |
| `yarn.log.server.url`                                  | Configura a URL base para acessar os logs agregados das aplicações usada redirecionamento do log do ResourceManager para o JobHistory Server.                                       |
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

#### 8.7 - Exportação dos paths:

Digite os comandos abaixo **em todas as máquinas** no usuário Hadoop para exportar os *PATHs* das aplicações:
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




### 11 - Formatar o Hadoop Directory File System - hdfs (main):

Digite o comando abaixo para carregar as variáveis de ambiente:
```bash
source /etc/environment
```
Digite o comando abaixo para realizar a formatação do hdfs:
```bash
hdfs namenode -format
```

> [!WARNING]
> É comum realizar a exclusão das pastas nameNode e dataNode para a correção de alguns possíveis erros, ou após alterações nos arquivos do hadoop, ambos os casos realizar uma nova formatação do hdfs é obrigatório para efetivar as alterações!




### 12 - Inicializar, monitorar e finalizar o cluster (main):

#### 12.1 - Inicializando o cluster:

Sempre que você desejar inicializar o *cluster*, você terá que primeiro carregar as variáveis de ambiente primeiro:  
```bash
source /etc/environment
```

Em seguida, digite o seguinte comando para inicializar o hdfs:
```bash
start-dfs.sh
```

Logo após, digite o seguinte comando para inicializar o yarn:
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

Para encerrar os serviços do *cluster*, basta substituir `start` por `stop` no comando.

Para verificar se o *cluster* foi inicializado corretamente, você pode digitar tanto na *main*, quanto nos *nodes*, o comando abaixo:  
```bash
jps
```


Esse comando deverá retornar algo semelhante às imagens abaixo:  
![Saida do JPS main][image1]  
![Saida do JPS node 1][image2]  
![Saida do JPS node 2][image3]

Se você estiver usando a máquina *main* como um *node*, também aparecerão os processos `DataNode` e `NodeManager`.

![Saida do JPS main como node][image4]

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

#### 14.1 - Limites para Aplicações nos nodes (via YARN):

Quando não configurado, o YARN assume valores padrões definidos no `yarn-default.xml`, o que pode não ser o ideal em um *cluster* com nodes que possuem baixa capacidade de recursos, como o cluster de *Raspberry Pi*, por exemplo. O hadoop também é capaz de detectar automaticamente os recursos das máquinas. Para realizar essa configuração, veja a sessão [15 - Configurando detecção de recursos](#15---configurando-detecção-de-recursos).

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
> Você pode consultar os valores padrão do yarn na [página oficial do yarn-default do Apache Hadoop](https://hadoop.apache.org/docs/r3.4.0/hadoop-yarn/hadoop-yarn-common/yarn-default.xml).  
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
> Você pode consultar os valores padrão do mapreduce na [página oficial do mapred-default do Apache Hadoop](https://hadoop.apache.org/docs/r3.4.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml).  
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

Caso apareça um erro sobre o limite de recursos ao rodar um trabalho, é sinal que os limites estão funcionando, mas é necessário entender como o YARN aloca realmente os recursos.

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

Caso não tenha definido os limites que uma aplicação pode utilizar, como mostrado na seção [14.1.2 - Configurando os limites de recursos para aplicações](#1411---configurando-os-limites-de-recursos-para-aplicações-nodes), vamos definir agora. 

Pode ser confuso pensar em definir limite se deveria ser algo automático, porém, o que definimos como automático é a detecção dos recursos da máquina, mas ainda é importante definir os limites mínimos e máximos que uma aplicação pode rodar nos *nodes* de forma que **mantenha a saúde** do nó e do *cluster*. A ideia é que o limite máximo não é mais um número arbitrário, mas sim um reflexo direto da capacidade real do seu hardware.

> [!NOTE]
> A configuração abordada aqui é mais simples do que a apresentada na seção [14.1.2 - Configurando os limites de recursos para aplicações](#1411---configurando-os-limites-de-recursos-para-aplicações-nodes), então recomendo fortemente que faça a leitura caso ainda não tenha feito.

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
| `-l 4G`       | Especifica o tamanho (*Length*) do arquivo. 2G para 2 Gigabytes. Você pode usar 4G, 8G, etc.  |
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

#### 16.2.1 - Defina as permissões do arquivo de Swap:

Por segurança, apenas o usuário root deve ter permissão para ler e escrever no arquivo de swap.
```bash
sudo chmod 600 /swapfile
```

#### 16.2.2 - Formate o arquivo de Swap e ative-o:
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

* [Instalando o Zabbix e Grafana para monitorar o cluster com Hadoop (em breve)](#)




### 18 - Realização de testes de benchmark para medir o desempenho do cluster:

* [Realizando testes de benchmark com o Hadoop (em breve)](#)




### 19 -  Automatizando o processo de configuração do cluster com Ansible:

* [Automatizando a configuração de servidores com Ansible (em breve)](#)

<!-- Imagens -->

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAM4AAABnCAYAAABIDH3iAAAS0ElEQVR4Xu2dP6hl1RXGU+QPo4aQ6WyEAVEIKCgEcQqDEQQHLCxsDCFWIaRJoZZi5RQhncHOThRSOY3NtGIngoWtVtoKqaxu8rvxN/lcb59z7zn3ztP73io+3j37rL322muvb/8/Mz+5evXqptFoLMNPakKj0diNJk6jsQJNnEZjBZo4jcYKNHEaB+PWrVubb7/9dvPoo4+eebcWzz333Jm0HxNmiXPryz/dwbWH7j/z/lD844Pnt0D/i395/Mz7y4ZT9cWxifPFF19s9b399ttb1Pc/BjRxvsNrb/1+895nL22BPe98/OK527TWFzdeemTz5vs3ttD+f95+YfPUjYfPyB4LkOSTTz7ZgiAH33zzzZZEVXYpmjg78GMhzt/+/rutDfwF2ELgkXY3g69ijS9oF4kO/vjKE5s/v3H9TrtV+WMhCfP5559vbt++vSUOzzdv3tyi5rlI2Js4Nio9Go2TcgZeNiBpVV8GJH9FBguB8Po7z25h70nZNYCVU2YklzbRG2u/BFHO96Yjp3zKVV0jf2BXtU270rb0RfojfTGqp53NY9evfU8Xz5mWJKxEzKB/9dVXv/duX5gfuB5hFFozQmADSJ0jApL21VdffW+kG41y2PPRRx9tZVMfsrXstdibOAQLDeBzNhRTBQKM6Y7EQYZn3htMNj6BlmTLYMmgpPEJGn6TN21TTpmRXOpHjjKSaDzbY6c9BLD6rEPq4z3pwLS0q9qmXdpWfVH9kYFufv7mSIIN6Y8KiUPbgHyXAfXuu++eybsPUkcN3KWAcEACOVUbEQcwwgHe5SinnPmRwbbEsdZhTZwmzpm8+6CJM0gUNhKQJHNTAJAkMVCUJR9BpWySLIPYvLVMgyzlRtMTp05pv9OkDD6CkaDid5IgyxgRJ6dcOb0yT7VNu9Q98oX+UFeWSTq6eU5iZt0T6R/bI99nkNa84Mpj/9r89JnN5udPfb259zc3N/c9+NctSOMdMgSqgQwI4Cl9S0HAzxHn5Zdf3oI0iE8aU7ORXNV9LOxNHJ0/RRwa0d4NGFDky7VDBiK6Up9B7HxfucwLUi7trWWM7Dev+UfEsV7V3qqr+qPqrnapr9o5pWtUz/TlqOMC1qXq34VfXvvDFhDkZ0//e0scfifueeStO/J1hACsParepdhFHEco0lwbYYNybFQkoV0HHWu0AQcTh/TsUStOkTi7RpzzJs4URsTBdsuvI80u1JHl1/c/vh1xfvHkp1tc+e2Hm189MD6YZFPgWLtqhxIHYE9uggCml7WstTiYODYU04ecOuSIM+o9SXdKpj6DdarMOlVLmSpX7Xd6pb2AqRrp/M4pHHrm1jijMvexH1B+9UX6Q12uIXkmXR8m0t92KuqdmsYBgi6nO8eCB6H0+KC+3xe7iJO2QxDScqqWkGS5bqoya3A04jAPpzFzoQ5oTPJmYOTaRth75uYAAZCLa/WknDJVrtqP3ZSR5Vomv8mfwbeGONpVbdOurEP1xciu1I+8I7dQxk5KOckpuSqJDl3YE4hOfwC9/nmOOLk5YF1SznMlbINQIElXy1uDg4lDOsFhwxs4SR6DhQY1YOzVlTNY1AeURR9BkLYpp8xILu2XENhp4FknAq8Sc4Tqi+oP7aq2aVfalr5If6Qu9WFr7WiQr3aNYGeQ9WAK49x/TYBzToIOgjZJyO+1u3SJXcRxRLPMun6p5zeSbc0Z0xSaOFebOLWeu9DE2UGcU0cG0BwhCFRknDLxXAO9MQ3XNsfYtXIaCHITAEiCmv5DoInzHRgB3M0zT64jGtM4lDiulfKazGix38Q5JywhTuOHg1M+RhmnVCNiNHEajRNHE6fRWIEmTqOxAk2cA5Fbp/Vd4+KiiXMgmjiXE7PE4SqFX9t5nYI7P/XaAodeBpBIObcb830F8u6UeMBlmehes4uS+r21m3YcI9jXECe/dtQ3+Mq0Q+55Nc4Hk8QhUGlEtwrZIvTbB5ByuY2IjHKkI5PEYX/e6+jeOTKAlLUMZH2ut1/3gXqqLaYtCfYpHEoc71KR37Qmzo8fk8QBNCbXK/LfuBpdh+B9HRGQgXCkJ3EyKDwhJt3RyWegTm+2zn18NUKSxjJGxKEcgjftoZ51ZEVeW/grqq7Up670o8ThvTqwS/n0Udrr9RI7maynHVx+i1+vmFRd6kuZUT3zikv12aie6skOouohT/XvKWGWOCNInFEQpzMzAJSlkXR8Tk1wKnlxus9AvTp76T8XlPppKEbCEXEMNkdA7XeUkgwGCDZkZ5C6sv78dfTNOuU3JObHBp+TOPqNtLyDlTKmqVN/gfRH1aU+ddV6YnvqAllP02o9LQ9do05CWcpP+04Ji4jj9A0n6GTfVQfvCvCUx/GkZUAJe0XS7bGqrimoHz0EJqSowZ51ylHBDoJ6OLryXL9wNPgMKHU72pKWH1Sh33qmDyjf56k6GtSi1hPYi1Nn7dK2qiv1kTZVzyRZ1tP0Ws86S9G27GxNyxg6JSwizlzDjjYI5shjw+pQ0ipxbNQsd1T2FNSPLnu5SpwsM/MmWXN0qeXn1LVOTaquKoeuDEptrGVINqBvkDPo1J1poyl11aU+803VM9ei1WejetYy07ZTJUpFE6eJcyffVD2bOGexN3H8RBVHzVVe5xgMI1nXMnWor2scnnNB745d1TcFG0xd/E5y1yDIvIcSZwqVOAI9vDNdP9bOKLGEOPvomqrnFHGm0MS5+v8zFSpuAFaZEdLR9V1+J+76RrgwBaZRLs/2klXfFNQjKSRzNjANOWpY7cdWy01dymovuiQnID17dlHXOGlvprsRwjNrB/1up6K9Wc9Mq8QZ6VKf+ZRVTpm6xrGezghGdRytcS4NcXSqTsogSucQ/L7LXos8I0e5iBztkuVuDXLqwpaljlePwZ66DYK0x6DMXTXLTJIYSElEdWXvjLz+EMgsJQ7l8K7ab/5M20Wc1JX6cpSznnUXUn2pH9R6Wp/sJNKWU96GFrPESYdVOBLh6HSiAUUw4rSqE3mdPSICzzaqDVh7yX2hTRLH4MmGzDLtQS2z2k9+30sQgy91AQIoiaU8MvsQRz3oVw821YDXn/mMrkqckS71ZV6AvJ0meuxQaj0lSq2n9amkE7bHKWOWOI0GyNF1TQd2EdHEaZyB03FGlzyDYjSpM4TLiiZO4wyaOLvRxGmcQW5u8Nu1TE/T/o8mTqOxAk2cRmMFmjiNxgo0cS4o6jlOfd84DLPEYWfFgzIP/ji8qie/POeBpQ1W9SFXdYGUVabKjfTtQj14A9hZr4QsgXpGh7dLoB7rSpoHhmvqWtHEubuYJI4n0h5+cWru7V2QsrkL4/tRY3n67JWSDOi8i2ValVsa8ObDfoIybziANYGfedfkr3q8rUBaE+d00MQZ5JtD5l2Tv+pp4pwmJokDcHgN1lGD5AVPA2LUWDVg0ZX6UqbK1TL3gXogo2l5oMe0jbR6p0qimSftnIJ2eeYxpStty44mP5/IOuKDnAZrS06XkXfai0z+Tp9VXepb0hk1/odZ4oxgAHm6nO92EccLheSjER3NgCTJS4dVbmkvr54kDkFiOjpJk/ReXPVelpcwCVKQN32tvzD4fFZf6spLndrliJM3kdN3Eh2/p28cpfBHEoFOrLaD+szPX79t0o7qu8Y8FhHH6RsNNQri2mCj/Nnj2zMmAZWpcpWk+yDzj9JBrQMwGA3OUd5R/UdIXalPu6wrxEjiqF9/S8wc/SAJsvzOTQYgYZM4PGcnBfRzjzrLsIg49Jg4uV6HF7uIYw9HY6kLEAhVpsqlzL5Qzz7EIXByBCFPvq95dxEn9akr9WlX9vxJHPNW+3MqiG/yd7ZLnd6OdKlPuVqHxjT2Jo6LdBw/FTBTxMneM/Pb2KSjP2WqnDKgljsFbclgqVM1nufWL7WumV7f8bxrPVSJ4xqS5wz2u0WcKTRxlmEncTK4begqI6aIs28QpEyVGwXHLmhL6sp1AiOaHQK/cyOEPKRXcriecN2T73IHUH2pK/VVu5xagTpVy3x1qjbyGbLaKXEkJ+k5soq5dm2cRROnidPEWYFZ4mSD43AbSkfrbHecMigJdtIy8GxMgh9dLphJQzZlqpwyyu2DtB17DDjLxW6DncBVf9ajEtV36kAvMqkr9aWu1MfvKUJnp+PiHdm6qya50mfuQKYu9WX97bRE9V1jHrPEyQavyN5/bk6fQeB1Gt/ZS+a6Ja/cpNyStY2othAwBF8SkMAjLYONsjNIDdCUT9JbT9+lvqpLfdZLW5N06bMsT30gO5DaCdEelpnEQRckyboqX33XmMcscRqNxhhNnEZjBZo4jcYKNHEajRVo4jQaK9DEaTRWoInTaKxAE6fRWIFZ4nDI5oGbh2sc2tWrJn6ElYdy9YCv6uWAzgNQdKcuDxEtMw8E12J0SHsMvY3LiUnieLrtFQ5Op6c+nfYuWV7dmCKO997qyXXexTL9mMQZXQs6ht7G5cQkcYCXA/MCYL2uTprXYzIQR8ThvYSAiHmVReR9OAl5zAAfXUTlN51DXvXxPlfmxQ/1eo6ytZzGxcYscUaQOHnfC4KQlgQbESe/Sqx6RzhP4gBvHpM+Gu2sJzL1kmTtABoXG02cq02cxnIsIo7rHoIqp1kGXgZPEkdZ0pwSmQfwPAq88yZOfmPjeo6pmXlTruptXC4sIo4L+/xGhSBKkohMc3Qy8AhennlvL17XE+C8iZOdgfZmvXJjw2v8PdpcTuxNHL8XIZAyUAg+0yWIQcfo4m5WBmIG5yhAxSHEYdqo/pxCHkIcgB/qiEk9a/mNi42dxNn16XQG4gjkcQpUg4zAlXijj6kOIU5+GJbTrdyOdo3mc07VzJ95E5KM+q21sXG6mCWOQQHcQs5evJIoIWlG+iAigcazaawpkMkRyiB3apcjwi4gl19F5j/D5NmUuqxjbg6MNjJ4h+3Y4yFtkq7a0Li4mCWOQTFCfjo9AjKVOBCt3kIABLRBPDrhF45QtawpOILk1MrPjh1ttDXrxG+IU9cv9fwGYO+az7obp40mzne2Zp343cRpzGGWOJcFkmDJVLBxudHEudrEaSxHE+dqE6exHE2cRmMFmjiNxgo0cRoHw4PqyzTNnSXOrS//dAfXHrr/zPtD8Y8Pnt8C/S/+5fEz7y8bTtUXTZyCJs754lR90cQpuEzEee2t32/e++ylLbDnnY9fPHeb1vrixkuPbN58/8YW2v/P2y9snrrx8BnZYwGS5BezYPTV7EVFE+e/+Nvff7e1gb8AWwg80u5m8FWs8QXtItHBH195YvPnN67fabcqfywkYbzDlx//LbnhcYrYmzg2Kj0ajZNyBl42IGlVXwYkf0UGC4Hw+jvPbmHvSdk1gJVTZiSXNtEba78EUc73piOnfMpVXSN/YFe1TbvStvRF+iN9Maqnnc1j1699TxfPmZYkrETMoM+rR0tgfuBlX0ahy3L9aG/iECw0gM/ZUEwVCDCmOxIHGZ55bzDZ+ARaki2DJYOSxido+E3etE05ZUZyqR85ykii8WyPnfYQwOqzDqmP96QD09Kuapt2aVv1RfVHBrr5+ZsjCTakPyokDm0D8l3eufNW+lKkjssyPUvsTRxJMteTgSSJgaIs+QgqZZNkGcTmrWUaZCk36mUdAdJ+e/sMPoKRoOJ3kiDLGBEnR44cJcxTbdMudY98oT/UlWWSjm6ek5hZ90T6x/bI94wM9aJr4spj/9r89JnN5udPfb259zc3N/c9+NctSOMdMn5HleubKX0XEXsTR+dPEYdGtHcDBhT5cgqUgei0Q30GsdMW5TIvSLm0t5Yxst+85h8Rx3pVe6uu6o+qu9qlvmrnlK5RPdOXo44LWJeqfxd+ee0PW0CQnz397y1x+J2455G37shLPr+pAkwDq96LiCZOE+cOmjj742DikJ5TkYpTJM6uqdp5E2cKI+Jgu+XXKdou1CnZr+9/fDtV+8WTn25x5bcfbn71wPirX/8fUsjTu2rRSFPEsaGYd+ecO0ecUe9JumsZ9RmsU2XWNU7KVLlqv+sS7QWscUjnd6590DO3OTAqcx/7AeVXX6Q/1OXmC8+k68NE+ttORb1T6x/AGif/fYVjwYPQXR85XgQcjTgsYGnM3OECNCZ5MzByU0DYe+auGgGQu1LqSTllqly1H7spI8u1TH6TP4NvDXG0q9qmXVmH6ouRXakfeUduoYydlHKSU3JVEh26IwbxPOz034PoESdgY2SwVOKQTnDY8AZOksdgoUENGHt15QwW9QFl0UcQpG3KKTOSS/slBHYaeNaJwKvEHKH6ovpDu6pt2pW2pS/SH6lLfdhaOxrkq10j2BlkPfx8HawJcD+B9x89kYT8Xru9fWqYJc6pIwNojhAEKjL2/DzXQG9Mo++qXTDsSxzACOCmhHlyOtSYRhPngqGJcz5o4lwwLCFOo7EEF5o4jcbdQhOn0ViB/wAExa29/R94egAAAABJRU5ErkJggg==>

[image4]: <data:image/png;base64,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>

[image5]: <data:image/png;base64,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I4HN01byBEO6jxqEWeY6YsSrO6z22odygSoMrJRIQI46VC6UsXvr2rEt+Nx4gWbemvTracW7IwvYdDWGYjbD6VhJmf2LF3Jbox5liyRhU708a1ctI0nbFFPLbrQu8fmex+/ph793jWyk/3tOcpYIiIAI/FQCz/120W34TOJTirPcbxN/vKj0U1VZKiMC2V5AQle2HwICIAIiIAIiIAIikBUCErqyQlnKEAEREAEREAERyPYCErqy/RAQABEQAREQAREQgawQkNCVFcpShgiIgAiIgAiIQLYXkNCV7YeAAIiACIiACIiACGSFgISurFCWMkRABERABERABLK9gISubD8EBEAEREAEREAERCArBCR0ZYWylCECIiACIiACIpDtBSR0ZfshIAAiIAIiIAIiIAJZISChKyuUpQwREAEREAEREIFsLyChK9sPAQEQAREQAREQARHICgEJXVmhLGWIgAiIgAiIgAhkewEJXdl+CAiACIiACIiACIhAVghI6MoKZSlDBERABERABEQg2wtI6Mr2Q0AAREAEREAEREAEskJAQldWKEsZIiACIiACIiAC2V5AQle2HwICIAIiIAIiIAIikBUCErqyQlnKEAEREAEREAERyPYCErqy/RAQABEQAREQAREQgawQkNCVFcpShgiIgAiIgAiIQLYXkNCV7YeAAIiACIiACIiACGSFgISurFCWMkRABERABERABLK9gISubD8EBEAEREAEREAERCArBCR0ZYWylCECIiACIiACIpDtBSR0ZfshIAAiIAIiIAIiIAJZISChKyuUpQwREAEREAEREIFsLyChK9sPAQEQAREQAREQARHICgEJXVmhLGWIgAiIgAiIgAhke4EfFrru7JrB9kB1DHXU0lFjIyPQMC5Jh06tMNXR+FPouNCXPFcypISB5j/bISHHmH1YBYc+Df/Zcv6Ju4few2n+ZrrbOlMm35dOT89sZs8DA4b1t0b9D8tW8P75HdArhr7uP2z8T7Rd7ikCIiACIiAC/wGBHxq6zun3YEh90wwWRSJXdi/jUEhJxgyxRlv561r+W2fzqvIgWhfP9c+S/kdD11+jhbDS2ROrwaMoYar716fLGSIgAiIgAiIgAj9c4J8LXUBS+AsWeqymVs/x1DJT47HfMXyuPiA+MRW9guXp1KYeYVcPs+HAeZJyGFK7UXvql8vDyQO7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>
