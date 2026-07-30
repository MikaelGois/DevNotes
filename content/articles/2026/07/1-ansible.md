---
title: Automatizando a configuração de servidores com Ansible
type: docs
weight: 1
editURL: "https://devnotes.msglabs.com.br/articles/ansible/"
next: /articles/2025/08/2-dns-and-proxy-server-rockylinux
---

Gerir um cluster Hadoop manualmente é um excelente exercício didático, mas uma vez que a configuração desejada está definida, a automação da implementação de novos nós é essencial. O Ansible permite-nos transformar máquinas recém-instaladas em nós Hadoop funcionais com um único comando.

Este é o terceiro artigo de uma série de 4 artigos. Porém, as informações podem ser usadas em qualquer outro cenário.

Além disso, a implementação do Ansible para automatizar o cluster ocorreu em um momento posterior ao descrito no artigo [Criando um cluster de computadores com Apache Hadoop](/articles/2025/07/1-hadoop-cluster). Diferente de como descrito nesse artigo, onde a implementação do cluster inicialmente ocorreu como um trabalho da disciplina de Arquitetura de Computadores, o cenário no presente artigo ocorre em uma outra oportunidade de implementar o cluster novamente, onde as configurações utilizadas já eram mais claras e maduras, o que facilitou a implementação do cluster de uma forma geral, e a implementação do Ansible para a automação.

Dito isso, a máquina usada para o Ansible também irá atuar como um roteador/gateway de internet para o cluster Hadoop, contornando o problema de acesso a internet do cluster que estava em uma rede local isolada devido as limitações da rede cabeada da faculdade. Isso foi possível porque a máquina possui adaptador Wi-Fi, o que coloca o cluster sob as mesmas regras e limitações de acesso a internet que qualquer outro dispositivo usado por alunos.

Se a sua intenção é apenas implementar o Ansible, seja em um cluster ou em outro cenário, não precisa se preocupar com as configurações abordadas nesse artigo referente a outros componentes/funções, basta seguir as instruções para a instalação e configuração do Ansible, e a criação dos playbooks para a automação de tarefas conforme a sua necessidade.

Nesse artigo temos o seguinte cenário:
- Um cluster Hadoop composto por 3 máquinas, sendo uma delas a principal (main/master/NameNode) e os outros dois nós (nodes/DataNodes), um já configurado e funcional, e outro que será implementado com o Ansible.
- Um computador que atua como backoffice, onde o Zabbix, o Grafana e o Ansible estão instalados e a partir do qual monitoramos e gerenciamos o cluster. Essa máquina também atua como gateway de internet para o cluster, permitindo que os nós do cluster tenham acesso a internet através dela.

> [!NOTE]
> Apesar do cenário descrito acima, o presente artigo tem como foco a implementação do Ansible em si, juntamente com as configurações necessárias para que ele possa cumprir sua função. As instruções para a instalação e configuração do Ansible, bem como a criação dos playbooks, podem ser adaptadas para outros cenários e necessidades. Não será abordada a configuração detalhada do cluster Hadoop, nem do Zabbix e Grafana, pois isso já foi abordado em outros artigos da série, e o foco aqui é a automação com Ansible.

> [!WARNING]
> É necessário instalar de forma prévia o sistema operacional em todos os dispositivos que serão afetados pela execução dos playbooks do Ansible. Além disso, é necessário ter configurado a rede e o acesso SSH. Este artigo não cobre a instalação do sistema operacional nem a configuração da rede e do acesso SSH.

> [!IMPORTANT]
> **As etapas detalhadas do script de provisionamento `provisionar_no_hadoop.yaml` apresentado mais adiante neste artigo refletem a implementação executada por mim em laboratório.** Cada um dos grandes blocos que compõem o playbook (bootstrap da chave SSH, sincronização de tempo, instalação do Java e do Hadoop, configuração de *swap*, agente Zabbix, reinicialização, atualização do mestre e registro no Zabbix) é independente e foi escolhido conforme as necessidades específicas do meu projeto. **Você deve incluir, ajustar, reordenar ou remover blocos de acordo com o que deseja automatizar no seu próprio projeto** — o playbook é uma referência, não uma receita obrigatória.

É importante entender que o backoffice é apenas um ponto de gerenciamento e monitoramento para o cluster, ele não é um componente do cluster Hadoop, ou seja, ele não substitui a máquina main que tem como função controlar o cluster Hadoop em si. A máquina backoffice é dispensável para o funcionamento do cluster, o que ela nos permite é monitorar e gerenciar o cluster de forma mais eficiente e externa, sem adicionar carga a máquina main, que poderia acabar acumulando essas funções, mas sob o custo de sobrecarregar a máquina e afetar o desempenho do cluster.

Outros artigos da série:
* [Criando um cluster de computadores com Apache Hadoop](/articles/2025/07/1-hadoop-cluster)
* [Instalando o Zabbix e Grafana para monitorar o cluster com Hadoop](/articles/2025/08/1-zabbix-and-grafana)
* [Realizando testes de benchmark com o Hadoop (em breve)](#)

## O que é o Ansible e por que usá-lo?

O Ansible é uma ferramenta de automação de TI que não precisa de agentes (*agentless*) instalados nos nós de destino. Ele funciona via SSH, o que o torna ideal para gerir dispositivos de forma remota. No contexto de clusters Hadoop, o Ansible permite-nos:
- Automatizar a instalação e configuração do Hadoop.
- Gerir atualizações e *patches* de segurança.
- Facilitar a escalabilidade do cluster, adicionando novos nós rapidamente.

## Atualizando os repositórios e pacotes:

Antes de iniciar, é importante atualizar os repositórios e pacotes instalados no sistema. No nosso caso, estamos usando um sistema baseado em Debian/Ubuntu, então basta digitar o seguinte comando:

```bash
sudo apt update && sudo apt upgrade
```

## Passos necessários para a implementação do Ansible:

### 1 - Instalação do Ansible (No Backoffice):

A instalação do Ansible deve ser feita apenas na tua máquina de controle (Backoffice/Master).

#### 1.1 - Atualizando os repositórios e pacotes:

Antes de iniciar, é importante atualizar os repositórios do sistema. No nosso caso, estamos usando um sistema baseado em Debian/Ubuntu, então basta digitar o seguinte comando:

```bash
sudo apt update
```

Depois instale o pacote `software-properties-common`, que é necessário para adicionar novos repositórios:

```bash
sudo apt install software-properties-common -y
```

#### 1.2 - Adicionando o repositório oficial do Ansible:

Primeiro, adicione o repositório oficial do Ansible:

```bash
sudo add-apt-repository --yes --update ppa:ansible/ansible
```

#### 1.3 - Instalando o Ansible e o sshpass:

Agora, instale o Ansible e o `sshpass` (necessário para a autenticação baseada em senhas, que utilizaremos no primeiro acesso aos nós) com o seguinte comando:

```bash
sudo apt install ansible sshpass -y
```

#### 1.4 - Verificando a instalação:

Depois de finalizada a instalação, verifique a versão do Ansible para confirmar que tudo está correto:

```bash
ansible --version
```




### 2 - Estrutura do Projeto e Configuração Inicial:

Dentro do diretório do Ansible em `/etc/ansible/`, você deverá criar uma estrutura organizada para armazenar seus playbooks, inventários, templates e arquivos de configuração. Esse caminho é o padrão utilizado por este artigo e pelo cluster do laboratório; você pode usar outro diretório desde que ajuste os caminhos referenciados no `ansible.cfg` e nos playbooks.

#### 2.1 - Diretório do Projeto:

Exemplo de estrutura:
```bash
/etc/ansible/
├── files-to-send/           # Pasta com os arquivos a serem enviados aos nós
│   ├── hadoop-3.3.6.tar.gz  # Pacote do Hadoop a ser replicado
│   ├── hadoop-files/        # Arquivos de configuração do Hadoop (core-site.xml, yarn-site.xml, workers, etc.)
│   ├── .bashrc              # .bashrc customizado (opcional)
│   └── hosts                # Arquivo /etc/hosts modelo do cluster
├── host_vars/               # Onde ficarão as senhas criptografadas por nó
├── playbooks/               # Os scripts de automação (YAML)
├── templates/               # Templates Jinja2 (ex.: chrony.conf.client.j2)
├── .vault_pass              # Senha mestre para as credenciais do Vault
├── ansible.cfg              # Configurações de comportamento do Ansible
├── hosts.ini                # Inventário das máquinas geridas
├── hosts-pass.yaml          # Lista de hosts e senhas (entrada para o mk-host-vars.sh)
├── mk-host-vars.sh          # Script para gerar host_vars/ em massa
└── zabbix_secrets.yaml       # Vault com a senha da API do Zabbix (se for usar)
```

Se a pasta `/etc/ansible/` não existir, você pode criar ela e as subpastas necessárias com o comando abaixo:

```bash
sudo mkdir -p /etc/ansible/{host_vars,files-to-send/hadoop-files,playbooks,templates}
```

Para criar os arquivos necessários, você pode usar o comando `touch`:

```bash
sudo touch /etc/ansible/{ansible.cfg,hosts.ini,hosts-pass.yaml,.vault_pass,zabbix_secrets.yaml}
```

#### 2.2 - Configurar o ansible.cfg:

Crie ou acesse este arquivo para definir o comportamento padrão do Ansible (evitar ter de confirmar a chave SSH de cada nó manualmente, indicar o inventário, o arquivo de senha do Vault, o usuário remoto e a chave privada a usar):

```bash
sudo nano /etc/ansible/ansible.cfg
```

Adicione as configurações abaixo:

```ini {filename="ansible.cfg"}
[defaults]
inventory = hosts.ini
host_key_checking = False
vault_password_file = .vault_pass
remote_user = hadoop
private_key_file = /home/backoffice/.ssh/ansible
```

{{% details title="Explicação das diretivas (Clique para expandir)" closed="true" %}}

| Diretiva              | Função                                                                                                                                                                                                                |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `inventory`           | Caminho padrão do arquivo de inventário (`hosts.ini`), dispensando digitar `-i hosts.ini` em cada comando.                                                                                                            |
| `host_key_checking`   | Quando `False`, o Ansible não pergunta se confia na chave SSH do host remoto no primeiro acesso — essencial para automação desassistida.                                                                              |
| `vault_password_file` | Caminho do arquivo com a senha mestre do Vault, lido automaticamente para criptografar/descriptografar variáveis — dispensa digitar `--vault-password-file .vault_pass` em cada comando.                               |
| `remote_user`         | Usuário SSH usado para se conectar aos nós geridos (neste projeto, `hadoop`).                                                                                                                                         |
| `private_key_file`    | Caminho absoluto da chave privada SSH usada na autenticação por chave após o bootstrap ser concluído. Usamos o caminho absoluto `/home/backoffice/.ssh/ansible` (e não `~/.ssh/ansible`) porque o `~` expande para o *home* do usuário que executa o comando — ao rodar `ansible-playbook` com `sudo`, o `~` vira `/root` e a chave não é encontrada. |

{{% /details %}}

> [!WARNING]
> O caminho da `private_key_file` deve ser **absoluto** (`/home/backoffice/.ssh/ansible`), não relativo com `~`. O til (`~`) expande para o *home* do usuário que executa o comando: ao rodar `ansible-playbook` com `sudo`, o `~` vira `/root/.ssh/ansible` — e a chave não existe lá, gerando o erro `no such identity: /root/.ssh/ansible`. Com o caminho absoluto, o Ansible encontra a chave independentemente de executar com ou sem `sudo`.

> [!NOTE]
> O arquivo `ansible.cfg` instalado pelo pacote oficial (ou gerado por `ansible-config init`) vem cheio de diretivas comentadas como documentação de referência. Você precisa apenas garantir que as diretivas acima estejam descomentadas em `[defaults]`; o restante pode ser mantido como referência ou removido conforme preferir.

#### 2.3 - Inventário de Hosts (hosts.ini):

O inventário define os nós do cluster Hadoop e seus grupos. Crie ou edite o arquivo `hosts.ini`:

```bash
sudo nano /etc/ansible/hosts.ini
```

Adicione o seguinte conteúdo, ajustando os IPs para a sua rede:

```ini {filename="hosts.ini"}
[all:vars]
ansible_user=hadoop
ansible_python_interpreter=/usr/bin/python3

# 1. Grupo para a máquina principal (Master/NameNode)
# Adicione a máquina principal do seu cluster aqui.
[main]
main ansible_host=192.168.0.10

# 2. Grupo para as máquinas de trabalho (Workers/DataNodes)
# Adicione todas as máquinas de trabalho já configuradas aqui.
[nodes]
node1 ansible_host=192.168.0.11

# 3. Um grupo que contém todos os nós do Hadoop (main e nodes)
# Útil para tarefas que precisam ser executadas em todo o cluster.
[cluster:children]
main
nodes

# 4. Grupo para os NOVOS nós a serem provisionados
# Para cada novo nó, defina seu IP e a variável 'novo_hostname'
[novos_nos]
#node2 ansible_host=192.168.0.12 novo_hostname=node2
```

{{% details title="Explicação dos grupos (Clique para expandir)" closed="true" %}}

| Grupo       | Função                                                                                   |
| ----------- | ---------------------------------------------------------------------------------------- |
| `[all:vars]`| Variáveis aplicadas a todos os hosts — aqui fixamos o usuário SSH e o interpretador Python. |
| `[main]`    | Apenas a máquina principal (NameNode/master).                                          |
| `[nodes]`   | Máquinas de trabalho (DataNodes) já configuradas manualmente.                            |
| `[cluster]` | Grupo derivado (`children`) que engloba `main` e `nodes` — atinge todo o cluster.      |
| `[novos_nos]`| Máquinas a serem provisionadas pelo playbook. Devem definir `novo_hostname`.           |

{{% /details %}}

> [!TIP]
> Mantenha o(s) host(s) novos comentados até o momento de provisioná-los — assim você evita execuções acidentais do playbook sobre máquinas que ainda não estão prontas para receber as configurações.

#### 2.4 - Gerando e Enviando Chaves SSH (Recomendado):

O acesso por chaves SSH é o método mais prático e seguro para gerenciar nós no Ansible. Caso opte por essa abordagem, primeiro gere um par de chaves no seu backoffice especificando o nome do arquivo da chave como `ansible`:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible
```

> [!NOTE]
> Pressione `Enter` para todas as opções sugeridas pelo comando, evitando adicionar uma *passphrase* se desejar execuções totalmente automatizadas pelo Ansible.

Depois, distribua a chave pública para os nós do inventário atual (por exemplo, a `main` e o `node1`) que ainda necessitam de autenticação, para permitir o acesso posterior sem senha de forma automatizada:

```bash
ssh-copy-id -i ~/.ssh/ansible.pub hadoop@192.168.0.10
ssh-copy-id -i ~/.ssh/ansible.pub hadoop@192.168.0.11
```

Ao executar o comando acima, a senha do usuário `hadoop` do nó de destino será requisitada pela primeira e única vez. Para os **novos** nós, as chaves SSH públicas do backoffice e do mestre serão enviadas automaticamente pelo playbook de provisionamento — veremos isso mais adiante.

#### 2.5 - Configuração do Vault para Armazenar Credenciais:

O Ansible Vault permite criptografar dados sensíveis, como senhas de sudo dos nós, para que não fiquem expostos em texto plano no projeto. Neste projeto, optamos por armazenar apenas a senha de escalonamento de privilégios (`ansible_become_password`) de cada nó, já que o acesso SSH é feito por chave pública após o bootstrap. Assim, o Ansible consegue executar comandos com `sudo` de forma automatizada nos nós sem expor senhas.

Nosso fluxo é: definir a senha mestre do Vault em `.vault_pass`, listar as senhas de cada host em `hosts-pass.yaml`, e usar o script `mk-host-vars.sh` para gerar um arquivo `host_vars/<host>.yaml` criptografado para cada nó automáticamente.

##### 2.5.1 - Criando o arquivo de senha mestre:

O Ansible Vault oferece quatro comandos principais para gerenciar arquivos criptografados:

| Comando | Função |
| --- | --- |
| `ansible-vault create segredo.yml` | Cria um novo arquivo já criptografado — abre o editor para inserir o conteúdo, que é salvo criptografado ao sair. |
| `ansible-vault edit segredo.yml` | Abre o arquivo criptografado no editor para edição, sem necessidade de descriptografá-lo manualmente. |
| `ansible-vault encrypt vars.yml` | Criptografa um arquivo já existente em texto plano, transformando-o em um arquivo Vault. |
| `ansible-vault decrypt segredo.yml` | Remove a criptografia do arquivo, voltando-o ao texto plano (não recomendado em produção). |

Neste projeto, optamos por usar `ansible-vault encrypt` em vez de `ansible-vault create` porque precisamos gerar os arquivos de credenciais em massa via script (`mk-host-vars.sh`), o que não é possível com o comando `create` (que abre um editor interativo). Para arquivos individuais como `zabbix_secrets.yaml`, o comando `create` é a abordagem recomendada.

Primeiro, defina a senha mestra no arquivo `.vault_pass` que será responsável por criptografar e descriptografar os arquivos:

```bash
echo "sua_senha_super_secreta" | sudo tee /etc/ansible/.vault_pass
```

> [!WARNING]
> Mantenha este arquivo seguro e evite versioná-lo em sistemas como os do Git, pois quem tiver esta senha poderá acessar todas as credenciais criptografadas do projeto. Recomendamos restringir o acesso a ela apenas para o usuário que executará o Ansible com `sudo chmod 600 /etc/ansible/.vault_pass`.

##### 2.5.2 - Criando o arquivo de lista de senhas (hosts-pass.yaml):

Crie um arquivo com os hosts e as senhas correspondentes de sudo do usuário `hadoop` em cada máquina:

```bash
sudo nano /etc/ansible/hosts-pass.yaml
```

Adicione os hosts e suas senhas no formato `hostname senha`, uma por linha:

```yaml {filename="hosts-pass.yaml"}
main SUA_SENHA_DO_HADOOP
node1 SUA_SENHA_DO_HADOOP
node2 SUA_SENHA_DO_HADOOP
# ... adicione aqui todos os hosts
```

> [!CAUTION]
> Esse arquivo contém as senhas em **texto plano** e existe apenas como entrada temporária para o script `mk-host-vars.sh`. **Não o versione em Git.** Após gerar os arquivos criptografados em `host_vars/`, você pode apagá-lo ou movê-lo para fora do projeto.

##### 2.5.3 - Criando o script para gerar host_vars/ em massa:

Para gerar em massa os arquivos de variáveis criptografados dentro de `host_vars/`, criaremos o script `mk-host-vars.sh`:

```bash
sudo nano /etc/ansible/mk-host-vars.sh
```

Adicione o seguinte conteúdo:

```bash {filename="mk-host-vars.sh"}
#!/bin/bash

# Arquivo de origem com os dados dos hosts (hostname senha)
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
    output_file="host_vars/${hostname}.yaml"

    # Cria o conteúdo YAML com a senha de sudo do host
    # e o criptografa usando ansible-vault, lendo do stdin.
    # Não passamos --vault-password-file aqui: o ansible.cfg já define
    # vault_password_file = .vault_pass, e passar a flag explicitamente
    # faria o ansible-vault enxergar dois vault-ids "default" (um do
    # ansible.cfg, outro da flag) e falhar com "default,default".
    printf -- 'ansible_become_password: "%s"\n' "$password" | \
    ansible-vault encrypt --output "$output_file"

done < "$ARQUIVO_SENHAS"

echo "Processo concluído. Arquivos criados em 'host_vars/'."
```

{{% details title="Por que apenas ansible_become_password? (Clique para expandir)" closed="true" %}}

| Variável                 | Quando é necessária                                                                                                                                               |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ansible_ssh_pass`        | Quando o acesso SSH é feito por senha. Como o Ansible usa chave SSH (enviada pelo `private_key_file` e pelo bootstrap da Etapa 1 do playbook), **não é necessária**.  |
| `ansible_become_password` | Senha usada pelo `sudo` para escalar privilégios no nó remoto (`become: true`). Como o usuário `hadoop` precisa de `sudo` para instalar pacotes, **é necessária**. |

Em projetos nos quais o acesso SSH ainda dependa exclusivamente de senha (sem chave pública), basta adicionar também `ansible_ssh_pass: "{{ password }}"` na linha do `printf` dentro do script — o Ansible passará a usá-la automaticamente.

{{% /details %}}

Dê permissões de execução para o script:

```bash
sudo chmod +x /etc/ansible/mk-host-vars.sh
```

Para gerar os arquivos criptografados para todos os hosts listados em `hosts-pass.yaml`, basta executar o script a partir do diretório `/etc/ansible`:

```bash
cd /etc/ansible && sudo ./mk-host-vars.sh
```

Ao final, cada host terá um arquivo `/etc/ansible/host_vars/<host>.yaml` criptografado. Quando o Ansible se conectar a esse host, ele usará o arquivo correspondente para obter a senha de `sudo` automaticamente, viabilizando o provisionamento de forma automatizada.

> [!IMPORTANT]
> Como o script foi executado com `sudo`, os arquivos `host_vars/*.yaml` gerados ficarão pertencentes ao usuário `root`. Se você pretende executar `ansible-playbook` **sem** `sudo` (aproveitando que a `private_key_file` agora usa caminho absoluto), ajuste o dono desses arquivos para o seu usuário, caso contrário o Ansible não conseguirá lê-los:
> ```bash
> sudo chown -R backoffice:backoffice /etc/ansible/host_vars/
> ```
> Se você sempre executar `ansible-playbook` com `sudo`, pode ignorar este passo — o `root` já tem acesso aos arquivos.

> [!NOTE]
> É necessário gerar esses arquivos para **todas** as máquinas do cluster, incluindo a máquina main, mesmo que ela já esteja configurada, para garantir que o Ansible tenha as credenciais necessárias para se conectar a todas as máquinas de forma automatizada.

> [!NOTE]
> Não precisamos especificar o caminho do arquivo de senha do Vault no script nem nos comandos `ansible-playbook`, porque o `ansible.cfg` já indica o caminho do arquivo de senha mestre com `vault_password_file = .vault_pass`, então o Ansible lê automáticamente a senha. O `ansible-vault encrypt` **herda** essa configuração do `ansible.cfg` — por isso o script não passa `--vault-password-file` explicitamente. Se passássemos a flag, o `ansible-vault` enxergaria dois vault-ids `default` (um do `ansible.cfg`, outro da flag) e abortaria com o erro `The vault-ids default,default are available to encrypt`.




### 3 - Pré-configuração do Cluster Hadoop:

Para realizarmos a implementação de novos nós Hadoop com o Ansible, é necessário que as máquinas estejam pré-configuradas com o sistema operacional, acesso SSH e rede configurados, e (se for o caso) o usuário `hadoop` criado. Além disso, precisamos de um padrão de configuração do Hadoop para ser replicado nos novos nós.

Tendo isso em mente, configuramos a máquina main (NameNode) e um dos nodes (DataNode) manualmente, seguindo as instruções do artigo [Criando um cluster de computadores com Apache Hadoop](/articles/2025/07/1-hadoop-cluster).

A máquina main tem configurações específicas para a sua função, enquanto as máquinas node têm configurações voltadas para o processamento de dados. A ideia é automatizar a implementação daquelas máquinas que, de uma forma geral, seguem o mesmo padrão de configuração — ou seja, os DataNodes. Por isso, também precisamos de um nó pré-configurado para servir como modelo para os novos nós a serem provisionados.

O que o Ansible irá fazer é pegar a configuração do node pré-configurado e replicá-la para os novos nós, garantindo que eles tenham as mesmas configurações e estejam prontos para se juntar ao cluster Hadoop.

Nada impede que o Ansible seja usado para configurar todo o cluster, mas esse processo adicionaria uma complexidade maior à configuração, havendo a necessidade de criar playbooks abrangendo a configuração de diferentes tipos de máquina e atendendo necessidades específicas de configuração para cada um dos tipos, o que seria uma configuração mais complexa e desnecessária para o nosso cenário, já que o foco é a automação da implementação dos DataNodes. Por isso, nesse cenário específico, optamos por configurar a máquina main manualmente e um dos nodes para ter um ponto de referência claro para as configurações dos DataNodes.

O que precisamos para realizar a replicação dos nós é:
- Arquivo `.pub` da chave SSH da máquina main (para comunicação interna do cluster Hadoop).
- Arquivo `.pub` da chave SSH do backoffice (`~/.ssh/ansible.pub`), usada pelo Ansible para autenticar nos nós via `private_key_file` (ver `ansible.cfg` na seção [2.2 - Configurar o ansible.cfg](#22---configurar-o-ansiblecfg)). Para os nós já existentes, essa chave é enviada por `ssh-copy-id` conforme a seção [2.4 - Gerando e Enviando Chaves SSH](#24---gerando-e-enviando-chaves-ssh-recomendado); para os **novos** nós, ela é enviada pela ETAPA 1 do playbook de provisionamento.
- Arquivo `.tar.gz` do Hadoop, que será enviado para os novos nós e descompactado lá, evitando a necessidade de baixar o Hadoop em cada nó.
- Arquivos de configuração do Hadoop (tudo dentro da pasta `/usr/local/hadoop/etc/hadoop/`), que serão replicados para os novos nós, garantindo que eles tenham as mesmas configurações e estejam prontos para se juntar ao cluster.
- (Opcional) Arquivo `.bashrc` customizado do usuário `hadoop`, com as variáveis de ambiente já definidas.
- (Opcional) Arquivo `/etc/hosts` modelo do cluster.

Esses arquivos ficarão armazenados na pasta `files-to-send/` e serão referenciados nos playbooks do Ansible para serem enviados e configurados nos novos nós.

Você pode utilizar os comandos abaixo para copiar os arquivos do nó pré-configurado para a pasta `files-to-send/`. Os IPs a seguir seguem a faixa utilizada no artigo [Criando um cluster de computadores com Apache Hadoop](/articles/2025/07/1-hadoop-cluster); ajuste conforme a sua rede:

```bash
sudo scp hadoop@192.168.0.10:/home/hadoop/.ssh/id_rsa.pub /etc/ansible/files-to-send/
sudo cp /home/backoffice/.ssh/ansible.pub /etc/ansible/files-to-send/
sudo scp hadoop@192.168.0.11:/home/hadoop/hadoop-3.3.6.tar.gz /etc/ansible/files-to-send/
sudo scp -r hadoop@192.168.0.11:/usr/local/hadoop/etc/hadoop/* /etc/ansible/files-to-send/hadoop-files/
sudo scp hadoop@192.168.0.11:/home/hadoop/.bashrc /etc/ansible/files-to-send/
```

Caso haja outras configurações que você precise replicar para os novos nós, como, por exemplo, arquivos de configuração do sistema, scripts de inicialização, etc., você pode usar o comando `scp` para copiar esses arquivos para a pasta `files-to-send/` e depois referenciá-los nos playbooks do Ansible para serem enviados e configurados nos novos nós.




### 4 - Configurando o Backoffice como gateway e servidor de horas para o cluster:

Como citado no início deste artigo, o cluster foi implementado em uma rede local isolada, sem acesso à internet, devido às limitações da rede cabeada da faculdade. Para contornar esse problema, o backoffice, que é a máquina onde o Ansible está instalado, foi configurado para atuar como um *gateway* de internet para o cluster Hadoop, permitindo que os nós do cluster tenham acesso à internet através dela.

Além disso, aproveitamos o backoffice para atuar também como servidor NTP (sincronização de horas) para o cluster. Isso evita problemas de dessincronia temporal entre os nós, que pode causar falhas no processamento do Hadoop, e funciona mesmo em redes isoladas sem acesso a servidores NTP externos.

#### 4.1 - Configurando IP estático:

Para configurar um IP estático na máquina backoffice (assumindo Ubuntu com Netplan), realize os passos abaixo.

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

As configurações de rede devem seguir o formato abaixo, onde você deve substituir as informações de acordo com a sua rede e interfaces. O backoffice precisa de duas interfaces de rede no nosso caso, visto que também servirá como gateway, será uma conectada à rede interna do cluster (com IP estático, `enp0s3` no exemplo) e outra conectada a uma rede com acesso à internet (via DHCP, `enp0s8` no exemplo). A interface com internet pode ser tanto uma interface cabeada (LAN) quanto uma interface Wi-Fi — basta ajustar o nome da interface (ex.: `wlan0` no lugar de `enp0s8`) dentro do bloco `ethernets:` conforme o seu hardware:

```yaml {filename="01-network.yaml"}
network:
    version: 2
    renderer: networkd
    ethernets:
        # Interface conectada à rede interna do cluster (IP estático)
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
        # Interface conectada à rede com acesso à internet (DHCP)
        # Mude o nome (enp0s8) pelo da sua interface real (ex.: wlan0 para Wi-Fi).
        enp0s8:
            dhcp4: true
# RESPEITE A INDENTAÇÃO!
# 'enp0s3': interface conectada à rede interna do cluster. Substitua pelo nome real.
# 'enp0s8': interface conectada à rede com internet. Substitua pelo nome real (ex.: wlan0 para Wi-Fi).
# 'addresses: 192.168.0.1/24' Define o endereço IP do backoffice na rede do cluster. É comum usar o .1 para o gateway.
# 'nameservers: search:' Define domínios de busca. Pode ser outro domínio, ex.: lab.local.
# 'nameservers: addresses:' Define endereços dos servidores DNS, ex.: 8.8.4.4 9.9.9.9 1.1.1.1
```

> [!NOTE]
> Caso decida utilizar o arquivo `50-cloud-init.yaml` é necessário desativar o **cloud init** conforme instruido nos comentários do início do arquivo.
> Você pode configurar outras faixas de IP, porém as máquinas só irão conseguir se comunicar se estiverem na mesma rede.

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

Para permitir que o tráfego passe da rede interna para a rede com internet, ative o IP Forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Para tornar essa alteração persistente após o reinício da máquina, edite o arquivo `/etc/sysctl.conf` e descomente ou adicione a seguinte linha:

```text
net.ipv4.ip_forward=1
```

#### 4.3 - Configurando o iptables para permitir o encaminhamento de pacotes:

> [!NOTE]
> Substitua `enp0s8` pelo nome da sua interface com acesso à internet e `enp0s3` pela interface conectada à rede interna do cluster em todos os comandos a seguir.

Digite o comando abaixo para habilitar a regra de NAT `MASQUERADE` no `iptables`, que disfarça o tráfego da rede interna com o IP do backoffice na interface com internet:
```bash
sudo iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE
```

Digite o comando para permitir o roteamento da rede interna para a interface com internet:
```bash
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -j ACCEPT
```

Digite o comando para permitir o tráfego de retorno da internet para a rede interna:
```bash
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -m state --state RELATED,ESTABLISHED -j ACCEPT
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

Para garantir que os nós do seu cluster Hadoop tenham a mesma sincronização temporal e evitar falhas no processamento por horários dessincronizados, usaremos o backoffice como um servidor NTP local. Dessa forma, as máquinas em redes isoladas poderão se guiar pelo seu nó Ansible, sem depender de servidores NTP externos.

Para fazer isso de forma automatizada, vamos criar um playbook exclusivo para sincronização. Esse playbook também é útil para regular clusters inteiros caso ocorra alguma dessincronia manual posterior:

```bash
sudo nano /etc/ansible/playbooks/sincronizar_tempo.yaml
```

O playbook é dividido em duas etapas:
- **Etapa 1**: configura a própria máquina Ansible (`localhost`) como servidor NTP.
- **Etapa 2**: configura todas as máquinas do cluster como clientes NTP apontando para o backoffice.

Insira o seguinte conteúdo:

```yaml {filename="sincronizar_tempo.yaml"}
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
    - name: Instalar o pacote chrony no servidor
      ansible.builtin.package:
        name: chrony
        state: present

    - name: Permitir que a rede do cluster acesse o servidor NTP
      ansible.builtin.lineinfile:
        path: "{{ chrony_conf_path }}"
        line: "allow {{ cluster_network }}"
        state: present

    - name: Configurar o servidor para servir tempo mesmo se não sincronizado (local stratum)
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
  hosts: all # Roda em todos os hosts do inventário
  become: true
  vars:
    # O IP da máquina Ansible será o nosso servidor NTP.
    # Usamos 'hostvars' para pegar o IP que o Ansible descobriu ao rodar no localhost.
    ntp_server_ip: "{{ hostvars['localhost']['ansible_default_ipv4']['address'] }}"
    chrony_conf_path: "{{ '/etc/chrony.conf' if ansible_os_family == 'RedHat' else '/etc/chrony/chrony.conf' }}"
    chrony_service_name: "{{ 'chronyd' if ansible_os_family == 'RedHat' else 'chrony' }}"

  tasks:
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

{{% details title="Explicação dos módulos e variáveis (Clique para expandir)" closed="true" %}}

| Elemento                              | Função                                                                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `hosts: localhost` + `connection: local` | Indica que as tarefas da Etapa 1 devem rodar na própria máquina de controle (backoffice), não via SSH.                                                                            |
| `ansible.builtin.package`             | Instala pacotes de forma idempotente em qualquer distribuição suportada (Debian, RedHat, etc.).                                                                                       |
| `ansible.builtin.lineinfile`          | Garante que uma determinada linha exista (ou não exista) em um arquivo — aqui usamos para inserir `allow`, `local stratum 10` e `server ...`, e para remover linhas `pool`/`server`. |
| `ansible.builtin.service`             | Garante que o serviço `chrony` esteja `started` e `enabled` (inicia no boot).                                                                                                         |
| `notify` / `handlers`                 | Quando uma tarefa altera algo (ex.: adiciona a linha `local stratum 10`), ela "notifica" um *handler* que só executa ao final da *play* — aqui, reinicia o `chrony`.                  |
| `hostvars['localhost']['ansible_default_ipv4']['address']` | Acessa o IP do `localhost` descoberto automaticamente pelo *setup* do Ansible e o repassa como `ntp_server_ip` para os clientes.                                |
| `ansible_os_family`                   | Variável de fatos usada para decidir entre caminhos/nomes diferentes em Debian (`chrony`, `/etc/chrony/chrony.conf`) e RedHat (`chronyd`, `/etc/chrony.conf`).                          |

{{% /details %}}

Rode esse playbook pela primeira vez para estabelecer a sincronia com os hardwares já presentes (como o `main` e o `node1`). Como a Etapa 1 roda em `localhost` e a Etapa 2 roda em `cluster`, limite a execução a esses dois grupos — não faz sentido rodar em `novos_nos`, que ainda não foram provisionados:

```bash
cd /etc/ansible && sudo ansible-playbook --limit cluster,localhost playbooks/sincronizar_tempo.yaml
```

> [!TIP]
> Se a sua rede for diferente de `192.168.0.0/24`, sobrescreva a variável na linha de comando sem editar o playbook:
> ```bash
> sudo ansible-playbook --limit cluster,localhost playbooks/sincronizar_tempo.yaml -e "cluster_network=10.0.0.0/24"
> ```

> [!NOTE]
> Alternativamente, em vez de editar o `chrony.conf` linha por linha, é possível reescrever todo o arquivo usando um *template* Jinja2. Essa abordagem é útil quando se quer um arquivo de configuração "limpo" e gerenciado inteiramente pelo Ansible, mas consome um pouco mais de cuidado para não sobrescrever configurações locais pré-existentes.




## Gerindo o Cluster com Ansible

Agora que o ambiente está preparado e já temos os arquivos base do Hadoop copiados para `files-to-send`, vamos criar o playbook principal, chamado `provisionar_no_hadoop.yaml`. Ele será responsável por transformar uma máquina "nua" (apenas com sistema operacional, rede e SSH) em um DataNode completo do cluster, incluindo o registro automático no Zabbix.

> [!NOTE]
> **Pré-requisitos do nó:** antes de executar o playbook, cada nó deve ter (1) IP estático configurado, (2) SSH instalado e acessível, e (3) o cache de pacotes atualizado (`sudo apt update`). Esses três pontos garantem que o Ansible consiga conectar e instalar os pacotes (chrony, Java, etc.) sem erros de cache desatualizado (ex.: `404 Not Found`).

> [!IMPORTANT]
> **Lembre-se: o playbook abaixo reflete a implementação executada por mim em laboratório.** Cada um dos grandes blocos que compõem o playbook (bootstrap da chave SSH, definição de hostname, sincronização NTP, instalação do Java, instalação/configuração do Hadoop, configuração de *swap*, instalação do Zabbix Agent, reinicialização, atualização do mestre e registro no Zabbix via API) foi escolhido conforme as necessidades específicas do meu projeto. Eles são **independentes e adaptáveis** — sinta-se livre para **incluir, ajustar, reordenar ou remover** qualquer um deles de acordo com o que faz sentido para o seu próprio projeto. Trate este playbook como uma **referência**, não como uma receita obrigatória.

### 1 - Visão geral do playbook

O playbook `provisionar_no_hadoop.yaml` é dividido em 4 etapas (*plays*) independentes, que rodam em alvos diferentes:

| Etapa  | Alvo (hosts) | Função                                                                                                                  |
| ------- | ------------ | ----------------------------------------------------------------------------------------------------------------------- |
| ETAPA 1 | `novos_nos`  | Bootstrap da chave SSH — copia a chave pública do mestre para o novo nó usando senha (uma única vez).                   |
| ETAPA 2 | `novos_nos`  | Provisionamento completo do nó (com `become: true`) — hostname, NTP, Java, Hadoop, swap, Zabbix Agent e reboot. **(Zabbix Agent e reboot são opcionais)** |
| ETAPA 3 | `cluster`     | Atualiza o `/etc/hosts` de todos os nós já existentes (`main` + `nodes`) com o IP/hostname do novo nó, e o arquivo `workers` apenas na `main` para que o mestre reconheça o novo DataNode. |
| ETAPA 4 | `localhost`  | Registra o novo nó no Zabbix via API REST, sem precisar de SSH até o servidor de monitoração. **(opcional)**                           |

> [!WARNING]
> **Ao adicionar novos nós ao cluster, pode ser necessário executar `hdfs namenode -format` novamente no mestre.** Esse comando apaga os metadados existentes do NameNode, o que significa que **todos os dados do HDFS serão perdidos** — o cluster volta ao estado "vazio". Antes de formatar, é preciso **limpar as pastas de dados** tanto do NameNode (no mestre) quanto dos DataNodes (em cada nó), definidas em `hdfs-site.xml` pelas propriedades `dfs.namenode.name.dir` e `dfs.datanode.data.dir`. Em laboratório isso costuma ser aceitável; em produção, avalie o impacto e faça *backup* antes de prosseguir.

Para facilitar o entendimento, vamos apresentar cada etapa separadamente, com explicações sobre os blocos. Você pode criar o arquivo do playbook com o seguinte comando e depois juntar todos os trechos abaixo na mesma ordem:

```bash
sudo nano /etc/ansible/playbooks/provisionar_no_hadoop.yaml
```

### 2 - ETAPA 1: Bootstrap da Chave SSH

A primeira etapa do playbook usa a senha (provida pelo Vault) apenas uma única vez, para instalar a chave SSH pública do mestre no `authorized_keys` do novo nó. A partir desse ponto, todas as etapas seguintes poderão usar a chave SSH.

```yaml {filename="provisionar_no_hadoop.yaml"}
# -----------------------------------------------------------------------------
# ETAPA 1: Bootstrap da Chave SSH (Usa Senha)
# -----------------------------------------------------------------------------
- name: ETAPA 1 - Bootstrap da Chave SSH
  hosts: novos_nos # Limitado pelo --limit novos_nos
  become: false
  gather_facts: false
  tasks:
    - name: Copiar a chave SSH pública do mestre para o novo nó
      ansible.posix.authorized_key:
        user: "{{ ansible_user }}"
        state: present
        key: "{{ lookup('file', '/etc/ansible/files-to-send/id_rsa.pub') }}"

    - name: Copiar a chave SSH pública do backoffice (Ansible) para o novo nó
      ansible.posix.authorized_key:
        user: "{{ ansible_user }}"
        state: present
        key: "{{ lookup('file', '/etc/ansible/files-to-send/ansible.pub') }}"

```

{{% details title="Explicação (Clique para expandir)" closed="true" %}}

| Elemento                              | Função                                                                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `become: false`                       | Esta etapa não escala privilégios — precisa apenas inserir uma linha no `~/.ssh/authorized_keys` do usuário remoto.                                                                                            |
| `gather_facts: false`                 | Não coleta fatos do nó nesta etapa, deixando a execução mais rápida.                                                                                                                  |
| `ansible.posix.authorized_key`         | Módulo que garante que uma chave pública esteja presente no `authorized_keys` do usuário remoto.                                                                                     |
| `lookup('file', '/etc/ansible/files-to-send/id_rsa.pub')` | *Lookup* que lê o conteúdo da chave pública do mestre (previamente copiada para `files-to-send/` conforme a seção [3 - Pré-configuração do Cluster Hadoop](#3---pré-configuração-do-cluster-hadoop)) e a envia para o novo nó.                          |
| `lookup('file', '/etc/ansible/files-to-send/ansible.pub')` | *Lookup* que lê o conteúdo da chave pública do backoffice (previamente copiada para `files-to-send/` conforme a seção [3 - Pré-configuração do Cluster Hadoop](#3---pré-configuração-do-cluster-hadoop)) e a envia para o novo nó, permitindo que o Ansible autentique via `private_key_file` sem senha. |

{{% /details %}}

> [!NOTE]
> No projeto do laboratório, enviamos duas chaves públicas para o novo nó: a do mestre (`/etc/ansible/files-to-send/id_rsa.pub`), copiada previamente da máquina `main` via `scp`, e a do backoffice (`/etc/ansible/files-to-send/ansible.pub`), copiada localmente no backoffice via `cp` — ambas conforme a seção [3 - Pré-configuração do Cluster Hadoop](#3---pré-configuração-do-cluster-hadoop). A chave do backoffice é essencial porque o `ansible.cfg` define `private_key_file = /home/backoffice/.ssh/ansible`; sem ela no `authorized_keys` do novo nó, o Ansible não conseguiria autenticar via SSH nas etapas seguintes. A chave do mestre, por sua vez, é necessária para a comunicação interna do cluster Hadoop (SSH *passwordless* entre o `main` e os *workers*). Essa escolha não é obrigatória; você pode enviar quantas chaves públicas quiser, replicando a tarefa `loop`-ando sobre uma lista de arquivos `.pub`.




### 3 - ETAPA 2: Cabeçalho, variáveis, hostname e `/etc/hosts` do nó

> [!NOTE]
> **Esta etapa é opcional.** A instalação do Zabbix Agent e o registro do nó no Zabbix via API (ETAPA 4) são etapas opcionais — se você não utiliza Zabbix, pode remover os blocos correspondentes do playbook sem afetar o provisionamento do Hadoop.

A Etapa 2 é o coração do playbook. Todas as operações de instalação e configuração do novo nó ocorrem aqui, em sequência. Inicie pela definição das variáveis, coleta de fatos do `localhost` (necessária para resolver o IP do backoffice como servidor NTP) e definição do hostname e do bloco de hosts:

```yaml {filename="provisionar_no_hadoop.yaml"}
# -----------------------------------------------------------------------------
# ETAPA 2: Provisionamento Completo do Nó (Usa Chave SSH)
# -----------------------------------------------------------------------------
- name: ETAPA 2 - Configurar o Novo Nó
  hosts: novos_nos # Limitado pelo --limit novos_nos
  become: true
  serial: 1
  vars:
    main_ip: "192.168.0.10"               # IP da máquina main (NameNode)
    zabbix_server_ip: "192.168.0.1"       # IP da máquina do Zabbix (backoffice). Comente se não for usar.
    hadoop_user: "hadoop"                  # Usuário criado em cada nó para rodar o Hadoop
    hadoop_archive: "hadoop-3.3.6.tar.gz" # Nome do pacote .tar.gz do Hadoop a ser enviado
    hadoop_extracted_dir: "hadoop-3.3.6"  # Nome do diretório extraído do pacote
    hadoop_install_dir: "/usr/local/hadoop" # Diretório onde o Hadoop será instalado
    chrony_conf_path: "{{ '/etc/chrony/chrony.conf' if ansible_os_family == 'Debian' else '/etc/chrony.conf' }}"
    chrony_service_name: "{{ 'chrony' if ansible_os_family == 'Debian' else 'chronyd' }}"
  tasks:
    - name: Coletar fatos da máquina Ansible (localhost)
      ansible.builtin.setup:
      delegate_to: localhost
      delegate_facts: true

    - name: Definir o hostname da máquina
      ansible.builtin.hostname:
        name: "{{ novo_hostname }}"

    - name: Limpar o /etc/hosts antes de adicionar as entradas do cluster
      ansible.builtin.copy:
        dest: /etc/hosts
        content: |
          127.0.0.1 localhost
          ::1 localhost
        owner: root
        group: root
        mode: '0644'

    - name: Garantir que a linha do próprio nó exista em /etc/hosts
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "{{ ansible_host }} {{ novo_hostname }}"
        state: present

    - name: Garantir que a linha do nó mestre (main) exista em /etc/hosts
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "{{ main_ip }} main"
        state: present

    - name: Garantir que as linhas dos nós já existentes do cluster existam em /etc/hosts
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "{{ hostvars[item]['ansible_host'] }} {{ item }}"
        state: present
      loop: "{{ groups['nodes'] }}"
      when: hostvars[item]['ansible_host'] is defined
```

{{% details title="Explicação das variáveis e módulos (Clique para expandir)" closed="true" %}}

| Elemento                              | Função                                                                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `become: true`                        | As tarefas seguintes exigem `sudo` (instalação de pacotes, edição de `/etc/hosts`, etc.) — a senha é lida automaticamente da variável criptografada `ansible_become_password` em `host_vars/`. |
| `serial: 1`                           | Provisiona **um nó por vez**, em vez de em paralelo — útil para evitar sobrecarga nos downloads de pacotes (para o APT) e para que os *logs* fiquem legíveis.                          |
| `main_ip` / `zabbix_server_ip`        | Centralizam os IPs referenciados ao longo da Etapa 2 — altere em um único lugar se a rede mudar.                                                                                       |
| `hadoop_archive`, `hadoop_extracted_dir`, `hadoop_install_dir` | Permitem trocar a versão do Hadoop e o diretório de instalação sem editar o resto do playbook.                                                                              |
| `delegate_to: localhost`              | Tarefa excepcional — ela é "deslocada" (delegada) para rodar na própria máquina de controle Ansible (não no nó remoto), coletando seus fatos de rede. Sem ela, as etapas de NTP não teriam como saber o IP do backoffice. |
| `delegate_facts: true`                | Permite que os fatos coletados no `localhost` fiquem disponíveis para as demais tarefas da *play* via `hostvars['localhost']`.                                                       |
| `ansible.builtin.hostname`             | Define o *system hostname* persistente do novo nó usando a variável `novo_hostname` declarada no `hosts.ini`.                                                                          |
| `ansible.builtin.copy` (limpeza)       | **Sobrescreve** o `/etc/hosts` inteiro com apenas as entradas `localhost` (`127.0.0.1` e `::1`) antes de inserir as linhas do cluster — garante que o arquivo esteja limpo, sem resquícios de configurações anteriores do nó. |
| `ansible.builtin.lineinfile`           | Garante que cada linha de host exista em `/etc/hosts` de forma idempotente (`state: present`) — adiciona a linha apenas se ela ainda não estiver presente.                              |
| `loop: "{{ groups['nodes'] }}"`        | Itera sobre todos os nós já existentes no grupo `[nodes]` do inventário, adicionando suas respectivas entradas (`IP hostname`) ao `/etc/hosts` do novo nó.                              |
| `hostvars[item]['ansible_host']`       | Recupera o IP (`ansible_host`) de cada nó diretamente do `hosts.ini` — não é preciso codificar IPs no playbook, basta manter o inventário atualizado.                                  |

{{% /details %}}

> [!NOTE]
> A tarefa `Limpar o /etc/hosts` sobrescreve o arquivo inteiro com apenas as entradas `localhost` (`127.0.0.1` e `::1`). Isso é necessário porque, ao provisionar um novo nó (por exemplo `node3`), o `/etc/hosts` pode conter entradas antigas ou resquícios de configurações anteriores que conflitam com as entradas do cluster. Após a limpeza, as tarefas `lineinfile` seguintes reescrevem o arquivo com as entradas corretas — o próprio nó, o nó mestre (`main`) e todos os nós já existentes.

> [!TIP]
> Como os IPs são lidos diretamente do `hosts.ini` via `hostvars[item]['ansible_host']`, não é preciso ajustar nenhum `range` ou soma no playbook — basta manter o inventário atualizado. Quando um novo nó for provisionado, mova-o do grupo `[novos_nos]` para `[nodes]` e execute o playbook novamente para que todos os nós atualizem seus `/etc/hosts`.


### 4 - ETAPA 2 (cont.): Sincronização NTP e instalação do Java e do SSH

Logo após o hostname ser definido, sincronizamos o relógio do nó com o backoffice (que configuramos como servidor NTP na seção anterior) e instalamos o OpenJDK 11, que é pré-requisito do Hadoop. Garantimos também que o serviço SSH continue ativo e habilitado:

```yaml {filename="provisionar_no_hadoop.yaml"}
    - name: Configurar e sincronizar NTP
      block:
        - name: Definir o fuso horário para America/Sao_Paulo (UTC-3)
          timezone:
            name: America/Sao_Paulo
        - name: Tentar atualizar o cache do apt (ignora erros)
          ansible.builtin.apt:
            update_cache: yes
          ignore_errors: true
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

{{% details title="Explicação de módulos (Clique para expandir)" closed="true" %}}

| Elemento                              | Função                                                                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `block:`                              | Agrupa tarefas relacionadas numa única unidade lógica, melhorando a leitura do playbook.                                                                                                |
| `chronyc waitsync 30 0.5`             | Comando do *chrony* que bloqueia até a sincronização ocorrer (até 30 amostras, com precisão de 0.5 segundos). Evita outras etapas rodarem com relógio dessincronizado.            |
| `changed_when: false`                 | Como essa tarefa é apenas uma leitura (*wait*), marcá-la como *changed=false* evita ruído na saída do Ansible (a tarefa fica sempre "ok" em vez de "changed").                       |
| `ignore_errors: true`                 | Permite que o `apt update_cache` falhe (ex.: internet instável) sem abortar toda a instalação — útil quando o cache local já é suficiente. Aplicado tanto antes de instalar o `chrony` quanto antes do OpenJDK 11, para evitar erros de cache `apt` desatualizado (ex.: `404 Not Found` ao buscar pacotes como `tzdata`).  |
| `ansible.builtin.apt`                  | Módulo específico para Debian/Ubuntu. Para provisões em RedHat/CentOS, troque por `ansible.builtin.dnf` ou use `ansible.builtin.package`.                                             |

{{% /details %}}




### 5 - ETAPA 2 (cont.): Instalação e Configuração do Hadoop

Neste bloco, o pacote `.tar.gz` do Hadoop (que está em `files-to-send/`) é enviado ao novo nó e descompactado em `/usr/local/hadoop`. Em seguida, os arquivos de configuração do Hadoop (a pasta `hadoop-files/`) e o `.bashrc` customizado são copiados, as variáveis de ambiente são adicionadas ao `.bashrc` e o diretório de dados do DataNode é criado:

```yaml {filename="provisionar_no_hadoop.yaml"}
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

        - name: Extrair o arquivo do Hadoop (se /usr/local/hadoop não existir)
          ansible.builtin.unarchive:
            src: "/home/{{ hadoop_user }}/{{ hadoop_archive }}"
            dest: "/home/{{ hadoop_user }}/"
            owner: "{{ hadoop_user }}"
            group: "{{ hadoop_user }}"
            remote_src: yes
            creates: "{{ hadoop_install_dir }}"

        - name: Mover e renomear o diretório do Hadoop para /usr/local/hadoop
          ansible.builtin.command: "mv /home/{{ hadoop_user }}/{{ hadoop_extracted_dir }} {{ hadoop_install_dir }}"
          args:
            creates: "{{ hadoop_install_dir }}"

        - name: Copiar arquivos de configuração do Hadoop para o nó
          ansible.builtin.copy:
            src: "/etc/ansible/files-to-send/hadoop-files/"
            dest: "{{ hadoop_install_dir }}/etc/hadoop/"
            owner: "{{ hadoop_user }}"
            group: "{{ hadoop_user }}"

        - name: Copiar o arquivo .bashrc customizado
          ansible.builtin.copy:
            src: "/etc/ansible/files-to-send/.bashrc"
            dest: "/home/{{ hadoop_user }}/.bashrc"
            owner: "{{ hadoop_user }}"
            group: "{{ hadoop_user }}"

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

        - name: Criar o diretório HDFS para o DataNode
          ansible.builtin.file:
            path: "{{ hadoop_install_dir }}/data/datanode" # Pode ser outro caminho, dependendo da configuração do hdfs-site.xml
            state: directory
            owner: "{{ hadoop_user }}"
            group: "{{ hadoop_user }}"
```

{{% details title="Explicação dos módulos (Clique para expandir)" closed="true" %}}

| Elemento                              | Função                                                                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ansible.builtin.stat` + `register`    | Verifica se o arquivo `.tar.gz` já existe no destino. A condição `when: not hadoop_archive_check.stat.exists` faz a cópia pular em reexecuções, economizando banda.                    |
| `ansible.builtin.copy`                | Copia um arquivo ou diretório do backoffice para o nó remoto. Aqui usamos para o `.tar.gz`, para a pasta inteira `hadoop-files/` e para o `.bashrc`.                                   |
| `ansible.builtin.unarchive`           | Descompacta um `.tar.gz`. O `remote_src: yes` indica que o arquivo já está no nó remoto (e não no backoffice). `creates: /usr/local/hadoop` evita descompactar de novo em reexecuções.            |
| `ansible.builtin.command`             | Executa um comando shell literal — aqui `mv hadoop-3.3.6 /usr/local/hadoop`. O `args.creates` evita rodá-lo se `/usr/local/hadoop` já existir.                                                            |
| `ansible.builtin.file` (state: directory) | Garante que o diretório `/usr/local/hadoop/data/datanode` exista com o dono `hadoop`.                                                                                                            |

{{% /details %}}

> [!WARNING]
> O caminho `{{ hadoop_install_dir }}/data/datanode` precisa estar em consonância com `dfs.datanode.data.dir` no `hdfs-site.xml` em `hadoop-files/`. Se você configurar o diretório de dados em outro lugar (por exemplo, em um cartão SD montado em `/mnt/hdfs-sdcard`), ajuste este bloco — ou remova-o — conforme a sua realidade.


### 6 - ETAPA 2 (cont.): Swap de 1G (opcional)

Muitos dos nós do laboratório possuem pouca memória RAM. Um *swap* de 1G estabelece uma "rede de segurança" para evitar travamentos durante picos de uso. Se o seu hardware tem bastante memória, este bloco pode ser removido:

```yaml {filename="provisionar_no_hadoop.yaml"}
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

> [!NOTE]
> Para entender melhor o que é *swap* e por que configurá-lo, consulte a seção [16 - Configurando o Swap](/articles/2025/07/1-hadoop-cluster#16---configurando-o-swap) do artigo sobre o cluster Hadoop. Aqui estamos apenas automatizando as etapas com Ansible.


### 7 - ETAPA 2 (cont.): Configuração do Zabbix Agent e reboot final

Neste bloco, o `zabbix-agent` é instalado e configurado em modo **ativo** — ou seja, é o agente que inicia a conexão com o Zabbix Server, e não o contrário. Isso é o que permite que os nós em redes isoladas (atrás do NAT do backoffice) ainda assim sejam monitorados. Ao final, o nó é reiniciado para aplicar o *hostname* e validar todas as configurações:

```yaml {filename="provisionar_no_hadoop.yaml"}
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

{{% details title="Explicação dos módulos (Clique para expandir)" closed="true" %}}

| Elemento                              | Função                                                                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `loop:` + `item`                      | Percorre a lista de pares `{key, value}` aplicando a mesma tarefa (`lineinfile`) para cada — aqui usamos para configurar tanto `ServerActive` quanto `Hostname` em poucas linhas.        |
| `notify: Reiniciar Zabbix Agent`     | Não reinicia imediatamente: apenas sinaliza o *handler* no final da *play*. Assim, se ambas as linhas forem alteradas, o `zabbix-agent` é reiniciado apenas uma vez ao final.           |
| `ansible.builtin.reboot`              | Reinicia o nó remoto via Ansible e espera ele voltar antes de prosseguir. O `post_reboot_delay: 30` dá uma folga de meio minuto para serviços estabilizarem.                          |
| `handlers:`                           | Seção do playbook reservada para tarefas que só rodam mediante `notify`. No nosso caso, o reinício do `zabbix-agent`.                                                                  |

{{% /details %}}

> [!WARNING]
> A instalação do `zabbix-agent` via `ansible.builtin.package` assume que o pacote já está disponível nos repositórios do nó. Em distribuições como **Debian/Ubuntu** "puros", pode ser necessário adicionar o repositório oficial do Zabbix previamente em cada nó (ou enviar um `.deb` pré-baixado no `files-to-send/` e instalá-lo via `ansible.builtin.apt` com `deb:`).


### 8 - ETAPA 3: Atualizar o arquivo `workers` na máquina main

Agora que o novo nó está de fato provisionado, precisamos avisar o restante do cluster sobre ele. São duas atualizações distintas:

1. **Adicionar a linha `<novo_ip> <novo_hostname>` ao `/etc/hosts` de todos os nós já existentes** (`main` + `nodes`) — assim o cluster consegue resolver o nome do novo nó. Usamos `lineinfile` (e não `blockinfile`) para que a linha seja inserida **uma única vez**, sem duplicar a cada execução.
2. **Adicionar o `<novo_hostname>` ao arquivo `workers` apenas na máquina `main`** — é esse arquivo que o Hadoop consulta para saber quais são os DataNodes. Por isso o `hosts:` dessa etapa é `main`, e não `cluster`.

> [!IMPORTANT]
> As tarefas abaixo rodam **apenas nos nós já existentes** (`main` e `nodes`), não nos `novos_nos` — afinal, o novo nó já recebeu o seu próprio `/etc/hosts` completo na [Etapa 2](#3---etapa-2-cabeçalho-variáveis-hostname-e-etchosts-do-nó). Repetir a operação em `novos_nos` seria redundante.

```yaml {filename="provisionar_no_hadoop.yaml"}
# -----------------------------------------------------------------------------
# ETAPA 3: Atualizar o Nó Mestre e o /etc/hosts dos nós existentes
# -----------------------------------------------------------------------------
- name: ETAPA 3 - Atualizar o Nó Mestre e o /etc/hosts dos nós existentes
  hosts: cluster # Apenas main + nodes (novos_nos NÃO estão em cluster)
  become: true
  tasks:
    - name: Garantir que a linha do novo nó exista em /etc/hosts
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "{{ hostvars[item]['ansible_host'] }} {{ item }}"
        state: present
      loop: "{{ groups['novos_nos'] }}"
      when: hostvars[item]['novo_hostname'] is defined

    - name: Adicionar os novos nós ao arquivo de workers (somente na main)
      ansible.builtin.lineinfile:
        path: "/usr/local/hadoop/etc/hadoop/workers" # Caminho do arquivo workers no mestre
        line: "{{ hostvars[item]['novo_hostname'] }}"
        state: present
      loop: "{{ groups['novos_nos'] }}"
      when:
        - inventory_hostname == 'main'
        - hostvars[item]['novo_hostname'] is defined

    - name: Lembrar de mover o nó provisionado para o grupo [nodes]
      ansible.builtin.debug:
        msg: "Provisionamento concluído! Agora mova o nó {{ hostvars[item]['novo_hostname'] }} do grupo [novos_nos] para [nodes] no arquivo hosts.ini e execute o playbook novamente para que todos os nós atualizem seus /etc/hosts."
      loop: "{{ groups['novos_nos'] }}"
      when:
        - inventory_hostname == 'main'
        - hostvars[item]['novo_hostname'] is defined
```

{{% details title="Explicação dos módulos (Clique para expandir)" closed="true" %}}

| Elemento                              | Função                                                                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `hosts: cluster`                      | Esta etapa roda apenas nos nós **já existentes** do cluster (`main` + `nodes`). Como `novos_nos` é um grupo separado, ele **não** é touched nesta etapa — evita duplicação desnecessária no `/etc/hosts` dos novos nós. |
| `ansible.builtin.lineinfile` (primeiro) | Garante que a linha `192.168.0.X node2` exista em `/etc/hosts` de cada nó existente. Idempotente: a linha só é inserida se ainda não existir — em reexecuções não duplica.                                       |
| `ansible.builtin.lineinfile` (segundo) | Garante que o hostname do novo nó esteja presente no arquivo `workers` da `main`. Idempotente: em reexecuções, o nome não é duplicado.                                       |
| `hostvars[item]['ansible_host']`        | Acessa o IP (`ansible_host`) declarado no `hosts.ini` para cada nó do grupo `novos_nos`. Repare como a linha é montada dinamicamente: `<ip> <hostname>`.                                  |
| `groups['novos_nos']`                  | Variável mágica do Ansible que lista todos os hosts do grupo `novos_nos` — ou seja, os novos nós provisionados nesta execução. Permite provisionar vários nós de uma vez.                 |
| `when: inventory_hostname == 'main'`   | Restringe a tarefa de atualizar o `workers` apenas à máquina `main`. Embora `hosts: cluster` percorra `main` e `nodes`, a condição garante que o `workers` só seja tocado pela `main`.                                  |
| `when: ... novo_hostname is defined`    | Protege caso a variável `novo_hostname` não esteja definida (ex.: nó comentado no inventário).                                  |
| `ansible.builtin.debug`                | Exibe uma mensagem de lembrete na saída do playbook, indicando que o nó provisionado deve ser movido do grupo `[novos_nos]` para `[nodes]` no `hosts.ini`. Restrito a `main` para que a mensagem apareça apenas uma vez. |

{{% /details %}}

> [!NOTE]
> Para que o mestre efetivamente reconheça o novo DataNode sem reiniciar todo o cluster, é necessário, após o playbook terminar, executar `hdfs dfsadmin -refreshNodes` na máquina main. Você pode automatizar isso também em uma quinta etapa no playbook, se preferir.


### 9 - ETAPA 4: Registrar o novo nó no Zabbix via API

> [!NOTE]
> **Esta etapa é opcional.** O registro do nó no Zabbix via API só é necessário se você utiliza o Zabbix como sistema de monitoramento. Caso contrário, pode remover esta etapa inteira do playbook sem impacto no provisionamento do Hadoop.

Por fim, registramos o novo nó automaticamente no Zabbix via sua API JSON-RPC. Esta etapa roda em `localhost` (no próprio backoffice, onde o servidor Zabbix está hospedado) e usa variáveis criptografadas em `zabbix_secrets.yaml` para a senha da API:

```yaml {filename="provisionar_no_hadoop.yaml"}
# -----------------------------------------------------------------------------
# ETAPA 4: Adicionar Host ao Zabbix via API
# -----------------------------------------------------------------------------
- name: ETAPA 4 - Adicionar Host ao Zabbix
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    zabbix_server_url: "http://192.168.0.1/zabbix/api_jsonrpc.php"
    zabbix_api_user: "ansible" # Substitua pelo usuário do Zabbix API
  vars_files:
    - /etc/ansible/zabbix_secrets.yaml # Arquivo vault com a senha do Zabbix API
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

    - name: 2. Obter ID do grupo 'Cluster'
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

    - name: 3. Obter ID do template 'Linux by Zabbix agent active'
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

{{% details title="Explicação dos módulos (Clique para expandir)" closed="true" %}}

| Elemento                              | Função                                                                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `hosts: localhost` + `connection: local` | As requisições à API do Zabbix são feitas a partir do próprio backoffice, que tem acesso direto ao servidor Zabbix — não há SSH envolvido.                                  |
| `vars_files: zabbix_secrets.yaml`       | Carrega a senha da API (`zabbix_api_password`) a partir de um arquivo criptografado pelo Vault — você deve criá-lo com `ansible-vault encrypt zabbix_secrets.yaml` contendo `zabbix_api_password: "SUA_SENHA_API"`. |
| `ansible.builtin.uri`                   | Módulo que faz requisições HTTP/HTTPS — usado para todas as 4 chamadas (`user.login`, `hostgroup.get`, `template.get`, `host.create`).                                                 |
| `register:`                             | Salva o *JSON* de resposta de cada requisição numa variável, para ser usada nas tarefas seguintes.                                                                                     |
| `ansible.builtin.assert`                | Verifica uma condição e interrompe com uma mensagem amigável caso falhe (ex.: grupo ou template não encontrados).                                                                      |
| `check_mode: no`                       | Força a executar mesmo em modo `--check` (dry-run), pois estas requisições não modificam o host remoto — apenas consultam/criam no Zabbix.                                            |
| `groups['novos_nos']`                  | Variável mágica do Ansible que lista todos os hosts do grupo `novos_nos`, garantindo que todos os novos nós provisionados nesta execução sejam registrados no Zabbix.                 |

{{% /details %}}

> [!NOTE]
> Antes de rodar esta etapa, certifique-se de que:
> 1. O grupo `Cluster` existe no Zabbix (crie-o manualmente na interface web se necessário).
> 2. O template `Linux by Zabbix agent active` existe no Zabbix (vem por padrão em instalações recentes do Zabbix 6.x+).
> 3. O usuário da API (`ansible` no exemplo) tem permissão para criar hosts e fazer login.
> 4. O arquivo `zabbix_secrets.yaml` foi criado e criptografado com `ansible-vault create`:
>    ```bash
>    sudo ansible-vault create /etc/ansible/zabbix_secrets.yaml
>    ```
>    No editor que se abrir, insira a variável da senha da API:
>    ```yaml {filename="zabbix_secrets.yaml"}
>    zabbix_api_password: SUA_SENHA_DA_API
>    ```
>    Salve e saia do editor — o arquivo já será salvo criptografado. Como o `ansible.cfg` já define `vault_password_file = .vault_pass`, o Ansible descriptografa o arquivo automaticamente durante a execução do playbook.

Com este playbook o Ansible fica encarregado de injetar configurações críticas do cluster em novos hardwares *bare-metal* e deixá-los atrelados ao registro de visualização sob o Zabbix, permitindo o provisionamento simultâneo de inúmeros nós.




## Executando o Playbook de provisionamento

Com todas as configurações e arquivos no lugar, você pode iniciar o provisionamento. Como as 4 etapas rodam em alvos diferentes (`novos_nos`, `cluster` e `localhost`), você precisa indicar todos no parâmetro `--limit`. Dentro do diretório `/etc/ansible`:

```bash
sudo ansible-playbook -i hosts.ini playbooks/provisionar_no_hadoop.yaml \
  --limit novos_nos,localhost,cluster --vault-password-file .vault_pass --ask-pass
```

{{% details title="Explicação das flags (Clique para expandir)" closed="true" %}}

| Flag                                         | Função                                                                                                                                                                                |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `-i hosts.ini`                               | Caminho do arquivo de inventário. Mesmo estando em `ansible.cfg`, é boa prática mantê-lo explícito nos exemplos.                                                                       |
| `--limit novos_nos,localhost,cluster`          | Restringe a execução aos hosts dos grupos `novos_nos` (alvos do provisionamento), `cluster` (atualização do `/etc/hosts` em todos os nós existentes e do `workers` na `main`) e `localhost` (registro no Zabbix). Sem isso, rodaria em todos os hosts. |
| `--vault-password-file .vault_pass`          | Indica qual arquivo contém a senha mestre do Vault. Embora o `ansible.cfg` já a aponte via `vault_password_file`, deixar explícito aqui ajuda em execuções com Vault alternativo.       |
| `--ask-pass`                                 | Solicita a senha SSH do usuário `hadoop` para que a Etapa 1 (bootstrap da chave) consiga de fato copiar a chave. Após o bootstrap, as etapas seguintes usam a chave SSH e a senha de `sudo` (do Vault). |

{{% /details %}}

> [!NOTE]
> Se você já configurou o `ansible.cfg` com `vault_password_file = .vault_pass` e não precisa de `-i hosts.ini` explícito, o comando mínimo fica:
> ```bash
> sudo ansible-playbook playbooks/provisionar_no_hadoop.yaml --limit novos_nos,localhost,cluster --ask-pass
> ```

### Retomando a execução a partir de uma tarefa específica

Se o playbook falhar ou for interrompido em algum ponto — ou se o nó já possui algumas configurações prévias — você pode iniciar a execução a partir de uma tarefa específica usando `--start-at-task`:

```bash
sudo ansible-playbook -i hosts.ini playbooks/provisionar_no_hadoop.yaml \
  --limit novos_nos,localhost,cluster --vault-password-file .vault_pass \
  --start-at-task "Instalação e Configuração do Hadoop"
```

O Ansible vai pular todas as tarefas anteriores e começar a executar a partir de `"Instalação e Configuração do Hadoop"`. Note que variáveis (`vars:`) e blocos `pre_tasks` ainda são processados normalmente — o `--start-at-task` apenas pula *tasks*, não *plays*.

### Executando apenas tarefas com uma tag específica

Para maior controle, você pode marcar tarefas ou *blocks* inteiros com tags no playbook (por exemplo: `tags: [hadoop]`, `tags: [ntp]`, `tags: [zabbix]`) e então rodar só aquela parte:

```bash
sudo ansible-playbook -i hosts.ini playbooks/provisionar_no_hadoop.yaml \
  --limit novos_nos,localhost,cluster --vault-password-file .vault_pass \
  --tags hadoop
```

### Variante: sem Vault configurado

Para comparação, em um cenário onde você ainda **não** usa o Vault (sem `ansible_become_password` armazenado), seria necessário pedir `-K` para a senha de `sudo`:

```bash
sudo ansible-playbook -i hosts.ini playbooks/provisionar_no_hadoop.yaml \
  --limit novos_nos,localhost,cluster --ask-pass -K
```

| Cenário                                                            | Comando                                                                                                                                                                  |
| ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Com Vault** (recomendado, usado no laboratório)                | `--vault-password-file .vault_pass --ask-pass`                                                                                                                            |
| **Sem Vault**, usando senha de sudo interativa                  | `--ask-pass -K` (pede senha SSH + senha sudo)                                                                                                                            |
| **Com chave SSH já no novo nó** (pós-bootstrap)                 | (omite `--ask-pass` — Ansible autentica pela chave em `private_key_file`)                                                                                              |

### Limitando a um subconjunto de hosts

Mesmo com o `--limit` apontando para os três grupos corretos, você pode querer provisionar apenas um nó específico, e não todos os do grupo `novos_nos`:

```bash
sudo ansible-playbook -i hosts.ini playbooks/provisionar_no_hadoop.yaml \
  --limit node2,localhost,cluster --vault-password-file .vault_pass --ask-pass
```

É possivel também combinar esses parâmetros de controle de execução (`--limit`, `--start-at-task` e `--tags`), permitindo que você execute apenas as partes necessárias, retome a execução a partir de um ponto específico ou execute a tarefa para um grupo específico de hosts, ou ainda uma combinação de tudo isso, o que pode economizar tempo e recursos, especialmente em playbooks mais longos ou complexos.




## Outros Playbooks Úteis

Além do playbook principal de provisionamento e do `sincronizar_tempo.yaml`, o repositório do laboratório contém outros playbooks que cobrem necessidades específicas. Nem todos são úteis em qualquer projeto, por isso a destacamos o `executar_comando.yaml` por ser essencial para operações em massa, e os demais são listados com uma breve descrição e o comando de uso, sem descer a código.

### 1 - Executando comandos genéricos em múltiplas máquinas (`executar_comando.yaml`)

Este é um dos playbooks mais úteis do dia a dia. Ele permite executar qualquer comando de terminal em um ou mais hosts do inventário, com ou sem `sudo`, sem precisar escrever um playbook novo a cada vez. O comando, o grupo alvo e a indicação de sudo são passados como *variáveis extras* na linha de comando (`-e`).

Crie o playbook:

```bash
sudo nano /etc/ansible/playbooks/executar_comando.yaml
```

Adicione o seguinte conteúdo:

```yaml {filename="executar_comando.yaml"}
- name: Executar comando genérico
  # O alvo da execução é definido pela variável 'anfitriao'.
  # Se não for definida, não executa em lugar nenhum por segurança.
  hosts: "{{ anfitriao | default('none') }}"
  # 'become' é ativado se a variável 'sudo' for 'true'.
  become: "{{ sudo | default(false) | bool }}"

  tasks:
    - name: "Executando comando: {{ comando }}"
      # O módulo 'shell' é mais flexível que 'command', pois permite pipes e redirecionamentos.
      ansible.builtin.shell:
        cmd: "{{ comando }}"
        # Faz com que o comando seja executado a partir de um shell de login,
        # o que pode ser importante para carregar variáveis de ambiente como as do Hadoop.
        executable: /bin/bash
      register: resultado_comando
      changed_when: false

    - name: Exibir resultado do comando
      ansible.builtin.debug:
        msg: "{{ resultado_comando.stdout_lines }}"
```

{{% details title="Explicação das variáveis dinâmicas (Clique para expandir)" closed="true" %}}

| Variável       | Função                                                                                                                                                                                |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `anfitriao`    | Define em qual grupo ou host do inventário o comando deve rodar. Se não for passada via `-e`, o valor padrão (`none`) evita a execução acidental em nenhum host — uma rede de segurança contra execuções fora do escopo. |
| `sudo`         | String convertida para `bool` — se `true`, ativa `become: true` (escala privilégios com `sudo`, pedindo a senha via Vault).                                                            |
| `comando`      | A string do comando shell a executar. Pode incluir pipes, redirecionamentos e variáveis (por isso usamos `shell` e não `command`).                                                    |
| `ansible.builtin.shell` com `executable: /bin/bash` | Ao setar o `bash` como shell efeito login, recarrega variáveis de ambiente do `.bashrc` (ex.: `PATH` do Hadoop), permitindo comandos como `hdfs dfs -ls /`.        |
| `changed_when: false`             | Comandos de leitura/consulta não devem ser marcados como "alterados" — esta flag some da saída qualquer alarme de mudança.                                               |

{{% /details %}}

#### Exemplos de uso:

```bash
# 1. Comando de leitura em todos os hosts do cluster (sem sudo):
sudo ansible-playbook -i hosts.ini playbooks/executar_comando.yaml \
  -e "comando='uptime'" -e "anfitriao=cluster" --vault-password-file .vault_pass

# 2. Atualizar o cache do apt em todos os nodes (com sudo):
sudo ansible-playbook -i hosts.ini playbooks/executar_comando.yaml \
  -e "comando='apt update'" -e "anfitriao=nodes" -e "sudo=true" \
  --vault-password-file .vault_pass

# 3. Desligar todos os nodes em massa (com sudo):
sudo ansible-playbook -i hosts.ini playbooks/executar_comando.yaml \
  -e "comando='sudo shutdown now'" -e "anfitriao=nodes" -e "sudo=true" \
  --vault-password-file .vault_pass

# 4. Listar arquivos HDFS a partir da máquina main:
sudo ansible-playbook -i hosts.ini playbooks/executar_comando.yaml \
  -e "comando='hdfs dfs -ls /'" -e "anfitriao=main" \
  --vault-password-file .vault_pass

# 5. Ver o status do HDFS em todos os nodes:
sudo ansible-playbook -i hosts.ini playbooks/executar_comando.yaml \
  -e "comando='hdfs dfsadmin -report'" -e "anfitriao=main" \
  --vault-password-file .vault_pass
```

> [!TIP]
> Para comandos ad-hoc ainda mais curtos (sem playbook), você pode usar o módulo `shell` diretamente:
> ```bash
> sudo ansible cluster -m shell -a "jps" --vault-password-file .vault_pass
> ```
> No entanto, o playbook `executar_comando.yaml` acima oferece vantagens: roda como *shell de login* (carregando variáveis do `.bashrc`) e imprime a saída *linha a linha* de forma legível através do módulo `debug`.

> [!WARNING]
> Cuidado com comandos de escrita/destrutivos. O `become: "{{ sudo | default(false) | bool }}"` protege contra escalonamento acidental, mas `rm -rf` ou `dd` com `sudo=true` são irrecuperáveis. Teste sempre em um único host com `--limit` antes de rodar contra o `cluster` inteiro.

