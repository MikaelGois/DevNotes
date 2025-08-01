---
title: Monitorando o cluster de computadores com o Zabbix e o Grafana
type: docs
weight: 1
editURL: "https://devnotes.msglabs.site/articles/zabbix-and-grafana/"
next: /articles/2025/07/1-hadoop-cluster
---

## Passos opcionais:

Os passos a seguir são opcionais, ou seja, não são essenciais para o funcionamento do cluster. 
Porém, monitoramento é algo essencial para qualquer sistema, portanto é recomendado a implementação.

> [!WARNING]
> Recomendável fazer as instalações, ou pelo menos baixar os pacotes, antes de configurar a rede (passo 6).
> Também é recomendável usar uma máquina a parte para realizar o monitoramento, ou seja, uma máquina que não faz parte do cluster. A ideia é evitar processamento na máquina `main`.

### 1 - Instalação e configuração do zabbix:

#### 1.1 - Instalação dos pacotes:

Baixe o pacote debian do zabbix, no meu caso estou usando a versão 6.0 e o Debian 11 (Bullseye):
```bash
sudo wget https://repo.zabbix.com/zabbix/6.0/debian/pool/main/z/zabbix-release/zabbix-release_latest_6.0+debian11_all.deb
```

Instale o pacote debian para adicionar o repositório do Zabbix:  
```bash
sudo dpkg -i zabbix-release_latest_6.0+debian11_all.deb
```

Atualize os repositórios:
```bash
sudo apt update
```

Para a versão do Zabbix com o Apache e banco de dados SQL, use o comando abaixo:
```bash
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

Para a versão do Zabbix com o Nginx e banco de dados SQL, use o comando abaixo:
```bash
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts zabbix-agent
```

No site do zabbix você pode encontrar mais informações sobre as versões disponíveis: [Zabbix Download](https://www.zabbix.com/download)


#### 1.2 - Instalação e configuração do banco de dados:

Uso o comando abaixo para verificar se o banco de dados ja está presente:
```bash
sudo dpkg -l | grep -E 'mysql-server|mariadb-server'
```

Se o retorno for `# ← Nenhum resultado`, instale o bando de dados com o comando a seguir:
```bash
sudo apt install mariadb-server
```

Verifique se o serviço subiu corretamente:
```bash
sudo systemctl status mariadb
```

Se não subir automaticamente, inicie manualmente:
```bash
sudo systemctl start mariadb
```

##### 1.2.1 - Crie o banco de dados inicial:

Acesse o MySQL:
```bash
mysql -uroot -p
```

Vai ser solicitado uma senha, então digite uma. Essa é a sua senha para acessar o MySQL. Exemplo.:
```sql
1q2w!Q@W
```

Depois crie o banco de dados propriamente dito:
```sql
create database zabbix character set utf8mb4 collate utf8mb4_bin;
```

Crie um usuário e senha para o banco de dados:
```sql
create user zabbix@localhost identified by 'SUA_SENHA';
```

> [!NOTE]
> No campo identificado como `'SUA_SENHA'`, você deverá adicionar uma senha, de preferência uma senha forte. Exemplo.: `create user zabbix@localhost identified by 'E5%MxK%qopVlx4'`

Adicione os últimos dois parâmetros:
```sql
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
```

Saia do MySQL:
```sql
quit;
```

##### 1.2.2 - Importação do esquema e dados iniciais:

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

##### 1.2.3 - Disabilite o log binário:
Isso permite que usuários criem funções armazenadas sem precisar de privilégios de superusuário.

Para desabilitar o log binário, acesse o MySQL novamente:
```bash
mysql -uroot -p
```

Digite a senha que você criou anteriormente, e execute o seguinte comando:
```sql
set global log_bin_trust_function_creators = 0;
```

Saia do MySQL:
```sql
quit;
```

#### 1.3 - Configuração do Zabbix:

##### 1.3.1 - Zabbix com Apache:
Acesse o arquivo de configuração do Zabbix:
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
Encontre a linha `DBPassword=` e adicione a senha que você criou para o usuário `zabbix` no MySQL:
```ini
DBPassword=SUASENHA
```
Salve e saia do editor.

Reinicie os serviços e habilite a inicialização no boot, para isso use o comando abaixo:
```bash
sudo systemctl enable --now zabbix-server zabbix-agent apache2
```

##### 1.3.2 - Zabbix com Nginx:
Acesse o arquivo de configuração do Zabbix:
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```

Encontre a linha `DBPassword=` e adicione a senha que você criou para o usuário `zabbix` no MySQL:
```ini
DBPassword=SUASENHA
```
Salve e saia do editor.

Em seguida, abra o arquivo de configuração do Nginx para configurar o PHP para o frontend do Zabbix:
```bash
sudo nano /etc/zabbix/nginx.conf
```

Descomente a linha `#listen 8080;` e `#server_name;` e as configure para habilitar o acesso via porta 8080:
```nginx
listen 8080;
server_name example.com; # Substitua pelo seu domínio
```
Salve e saia do editor.

Reinicie os serviços e habilite a inicialização no boot, para isso use o comando abaixo:
```bash
sudo systemctl enable --now zabbix-server zabbix-agent nginx php7.4-fpm
```


##### 1.3.3 - Acesso ao frontend do Zabbix:
Abra o navegador e acesse o endereço do Zabbix:
```bash
http://SEU_IP/zabbix
```

Se estiver usando o Nginx, acesse:
```bash
http://SEU_IP:8080/zabbix
```

O usuário padrão é `Admin` e a senha padrão é `zabbix`.

> [!WARNING]
> É recomendável alterar a senha do usuário Admin após o primeiro acesso.


Defina o idioma, o nome do host do servidor Zabbix e clique em `Next step`.

> [!NOTE]
> Caso deseje usar outro idioma que não esteja disponível, digite `sudo dpkg-reconfigure locales` no terminal e selecione o idioma desejado. Após isso, reinicie o serviço do Zabbix e acesse novamente o frontend.

Selecione o banco de dados MySQL, insira o nome do banco de dados (`zabbix`), o usuário (`zabbix`) e a senha que você criou anteriormente. Clique em `Next step`.

Siga as demais instruções na tela para concluir a configuração do Zabbix.

#### 1.4 - Configurações adicionais do Zabbix Server:

Após a instalação e configuração básica do Zabbix, você pode querer ajustar algumas configurações adicionais para otimizar o desempenho e a segurança do seu servidor Zabbix.
##### 1.4.1 - Ajuste de parâmetros de desempenho:
Abra o arquivo de configuração do Zabbix Server:
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```

Remova o comentário de `StartPollers` e `StartTrappers`  e ajuste os parâmetros conforme necessário:
```ini
StartPollers=5
StartTrappers=5
```

Esses parâmetros controlam o número de processos que o Zabbix Server usa para coletar dados de monitoramento e processar traps SNMP.

Caso você tenha um grande número de hosts ou itens monitorados, pode ser necessário aumentar esses valores para melhorar o desempenho do Zabbix Server. Para começar, você pode definir `StartPollers` e `StartTrappers` para 5.

Apos fazer as alterações, salve o arquivo e reinicie o serviço do Zabbix Server para aplicar as novas configurações:
```bash
sudo systemctl restart zabbix-server
```

##### 1.4.2 - Configuração de Timeout:

Para evitar problemas de timeout durante a coleta de dados, principalmente quando há hosts lentos ou de menor capacidade, é recomendável ajustar o parâmetro `Timeout` no arquivo de configuração do Zabbix Server.

Abra o arquivo de configuração do Zabbix Server:
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```

Remova o comentário de `Timeout` e ajuste o parâmetro conforme necessário:
```ini
Timeout=30
```

Esse parâmetro controla o tempo limite para as operações do Zabbix Server.

Após fazer as alterações, salve o arquivo e reinicie o serviço do Zabbix Server para aplicar as novas configurações:
```bash
sudo systemctl restart zabbix-server
```

#### 1.5 - Configuração do Zabbix Agent:

O Zabbix Agent é responsável por coletar dados dos hosts monitorados e enviá-los para o Zabbix Server. 

Para instalar o Zabbix Agent, digite o seguinte comando:
```bash
sudo apt install zabbix-agent
```

Inicie o serviço e habilite a inicialização automática no boot:
```bash
sudo systemctl enable --now zabbix-agent
```

##### 1.5.1 - Configuração do Zabbix Agent passivo:
Essa é a cocnfiguração padrão do Zabbix Agent, onde o agente aguarda solicitações do Zabbix Server para enviar dados.

Abra o arquivo de configuração do Zabbix Agent:
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```
Encontre a linha `Server=` e adicione o IP do Zabbix Server:
```ini
Server=SEU_IP_DO_ZABBIX_SERVER
```

Encontre a linha `ServerActive=` e comente-a, pois não usaremos o modo ativo neste exemplo:
```ini
# ServerActive=127.0.0.1
```

Encontre a linha `Hostname=` e adicione o nome do host do agente:
```ini
Hostname=SEU_NOME_DO_HOST
```

Salve e saia do editor.

> [!CAUTION]
> Certifique-se de que o nome do host no arquivo de configuração do agent seja igual ao usado no Zabbix Server e não conflite com outros hosts monitorados pelo Zabbix Server.

Reinicie o serviço do Zabbix Agent para aplicar as novas configurações:
```bash
sudo systemctl restart zabbix-agent
```

Agora é só adicionar o host no Zabbix Server, usando o nome do host que você definiu no arquivo de configuração do agente. Além disso, é necessário adicionar o template apropriado para coletar os dados do agente, por exemplo, `Linux by Zabbix agent`. Também é necessário adicionar e configurar a interface do agente para usar a porta TCP 10050, que é a porta padrão do Zabbix Agent.

##### 1.5.2 - Configuração do Zabbix Agent ativo:
No modo ativo, o Zabbix Agent envia dados para o Zabbix Server sem esperar por solicitações, o que pode ser útil quando se quer evitar que o Zabbix Server faça solicitações frequentes aos agentes. Pra máquinas mais lentas, o modo ativo pode ser mais eficiente.

> [!WARNING]
> O modo ativo evita que o Zabbix Server faça solicitações frequentes aos agentes, evitando sobrecarga de rede, mas pode aumentar a carga no Zabbix Server, especialmente se muitos agentes estiverem enviando dados ao mesmo tempo. Monitore o desempenho do servidor e ajuste as configurações conforme necessário. Além disso, é importante garantir que o host no Zabbix Server esteja configurado corretamente para receber dados do agente ativo usando um template apropriado, por exemplo, `Linux by Zabbix agent active`.

Abra o arquivo de configuração do Zabbix Agent:
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```
Encontre a linha `Server=` e comente-a, pois não usaremos o modo passivo neste exemplo:
```ini
# Server=SEU_IP_DO_ZABBIX_SERVER
```

Como o parâmentro `Server` é obrigatório, é necessário descomentar e definir o parâmetro `StartAgents` para `0`, o que faz com que o Zabbix Agent não escute as portas TCPs no host, em outras palavras, ele não ocupa a porta TCP 10050, que é a porta padrão do Zabbix Agent:
```ini
StartAgents=0
```

Descomente a linha `ServerActive=` e adicione o IP do Zabbix Server:
```ini
ServerActive=SEU_IP_DO_ZABBIX_SERVER
```

Encontre a linha `Hostname=` e adicione o nome do host do agente:
```ini
Hostname=SEU_NOME_DO_HOST
```
Salve e saia do editor.

> [!CAUTION]
> Certifique-se de que o nome do host no arquivo de configuração do agent seja igual ao usado no Zabbix Server e não conflite com outros hosts monitorados pelo Zabbix Server.

Reinicie o serviço do Zabbix Agent para aplicar as novas configurações:
```bash
sudo systemctl restart zabbix-agent
```

Agora é só adicionar o host no Zabbix Server, usando o nome do host que você definiu no arquivo de configuração do agente. Além disso, é necessário adicionar o template apropriado para coletar os dados do agente, por exemplo, `Linux by Zabbix agent active`. Não é necessário adicionar a interface do agente, pois o modo ativo não usa a porta TCP 10050, que é a porta padrão do Zabbix Agent.


### 2 - Instalação do Grafana:

O Grafana é uma plataforma de visualização e análise de dados que pode ser usada para criar painéis de controle (dashboards) e gráficos a partir dos dados coletados pelo Zabbix, ou de outras fontes, como por exemplo o Prometheus.

É opcional instalar o Grafana, mas é altamente recomendado para uma melhor visualização dos dados coletados pelo Zabbix.

#### 2.1 - Instalação e configuração dos pacotes:

Instale as dependências necessárias:
```bash
sudo apt install -y adduser libfontconfig1 musl apt-transport-https software-properties-common wget
```

Adicione a chave GPG do repositório do Grafana:
```bash
sudo mkdir -p /etc/apt/keyrings/ && sudo wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```

Para adicionar o repositório de versões estáveis do Grafana, execute o seguinte comando:
```bash
sudo echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

Se você quiser instalar versões Beta ou de desenvolvimento do Grafana, use o seguinte comando:
```bash
sudo echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com beta main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

Atualize os repositórios e instale o Grafana OSS:
```bash
sudo apt update && sudo apt install -y grafana
```

Se você desejar usar a versão Enterprise do Grafana, execute o seguinte comando:
```bash
sudo apt install grafana-enterprise
```

Reinicie o daemon e habilite o serviço do Grafana para iniciar automaticamente no boot:
```bash
sudo systemctl daemon-reload && sudo systemctl enable --now grafana-server
```

#### 2.2 - Acesso ao Grafana e integração com o Zabbix:

##### 2.2.1 - Acesso ao Grafana:
Após a instalação, o Grafana estará rodando na porta 3000 por padrão.

Abra o navegador e acesse o endereço do Grafana:
```bash
http://SEU_IP_DO_GRAFANA:3000
```

O usuário padrão é `admin` e a senha padrão é `admin`.

Após o primeiro acesso, você será solicitado a alterar a senha do usuário `admin`.

##### 2.2.2 - Configuração do usuário Grafana no Zabbix:
Por questões de segurança, é altamente recomendável criar um usuário específico no Zabbix para o Grafana, com permissões limitadas apenas para leitura dos dados necessários. A regra de ouro é o **Princípio do Mínimo Privilégio**: o usuário deve ter apenas as permissões estritamente necessárias para ler os dados que precisa, e nada mais.

###### 2.2.2.1 - Passo 1: Criar um Grupo de Usuários

1. No Zabbix, vá para **Administration** → **User groups**.

2. Clique em **Create user group**.

3. Dê um nome, como `Grafana Users`.

4. Vá para a aba **Permissions**. Aqui é onde você define o acesso aos dados.

5. Clique em **Add**. Selecione os **Host groups** que você deseja que o Grafana possa acessar. É crucial dar acesso apenas aos grupos de hosts que serão exibidos nos dashboards.

6. Defina a permissão como **Read**.

7. Clique em **Add** para salvar o grupo.

###### 2.2.2.2 - Passo 2: Criar uma Role (Função)
As Roles definem o que um usuário pode fazer na interface e na API.

1. Vá para **Administration** → **Roles**.

2. Clique em **Create role**.

3. Dê um nome, como `Grafana Role`.

4. Na aba **Permissions**, configure o seguinte:

   * **Type of users**: Selecione User.
   * **Access to UI elements**: Marque apenas o essencial para leitura. Uma boa base é:

      * **Monitoring** (para acesso a Latest data, Hosts, Problems, etc.)

   * **Access to API**:

      * Marque **Enabled** para `API access`. O plugin do Grafana precisa disso para funcionar.

   * **Access to actions**: Deixe tudo desmarcado. O Grafana não precisa executar ações.

5. Clique em **Add** para criar a role.

###### 2.2.2.3 - Passo 3: Criar o Usuário
Finalmente, crie o usuário que o Grafana usará.

1. Vá para **Administration** → **Users**.

2. Clique em **Create user**.

3. **Alias / Username**: Dê um nome, como `grafana_viewer`.

4. **Group**: Selecione o grupo que você criou (`Grafana Users`).

5. **Role**: Selecione a role que você criou (`Grafana Role`).

6. **Password**: Crie uma senha forte e segura.

7. Clique em **Add**.

Com este usuário (`grafana_viewer`), o Grafana poderá ler os dados dos grupos de hosts permitidos, mas não poderá fazer nenhuma alteração de configuração, reconhecer alertas ou acessar hosts que não foram explicitamente liberados.

##### 2.2.3 - Instalação do plugin do Zabbix no Grafana:
Para integrar o Grafana com o Zabbix, você precisará instalar o plugin que permite a comunicação do Grafana com o Zabbix

1. No Grafana, vá para o menu lateral esquerdo, clique em **Administration** → **Plugins and data** e depois em **Plugins**.

2. Clique no campo de busca **Search Grafana Plugins** e procure por **Zabbix**. Você encontrará o plugin `Zabbix` do Alexander Zobnin. Clique nele.

3. Clique em **Install** para instalar o plugin do Zabbix, aguarde finalizar a instalação e depois em **Enable**. A página será recarregada automaticamente.

> [!WARNING]
> Caso esteja usando uma versão mais antiga do Grafana, você pode precisar instalar o plugin manualmente. Para isso, digite no terminal o comando `grafana-cli plugins install alexanderzobnin-zabbix-app`.

> [!NOTE]
> Pode ser necessário reiniciar o Grafana para que o plugin funcione corretamente, faça isso com o comando `sudo systemctl restart grafana-server`.

##### 2.2.4 - Configurando datasource do Zabbix no Grafana:
Agora, você precisa dizer ao Grafana como se conectar à API do seu Zabbix.

1. No menu lateral direito do Grafana, vá para **Connections** → **Data sources**.

2. Clique no botão **Add new data source**.

3. Na lista de tipos de data source, procure e selecione **Zabbix**.

4. Você verá a página de configuração. Preencha os seguintes campos:

     * **Name**: Dê um nome para esta conexão, por exemplo, `Servidor Zabbix Principal`.

     * **URL**: Este é o campo mais importante. Você deve apontar para o arquivo api_jsonrpc.php do seu Zabbix. O formato é: `http://IP_DO_SEU_ZABBIX/zabbix/api_jsonrpc.php`
      (Substitua `IP_DO_SEU_ZABBIX` pelo IP ou domínio correto).

     * **Authentication**: Ative a opção Basic auth.

     * **User**: Digite o nome do usuário do Zabbix que você separou para esta integração.

     * **Password**: Digite a senha deste usuário.

Role para baixo e clique no botão **Save & test**. Se tudo estiver correto, você verá uma mensagem de sucesso como "Zabbix API version: 6.0.x". Se ocorrer um erro, verifique o URL, o usuário, a senha e as permissões do usuário no Zabbix.

##### 2.2.5 - Criando um Dashboard no Grafana:
Com a conexão funcionando, você já pode criar gráficos. A maneira mais fácil de começar é importando um dashboard pronto da comunidade.

1. Vá até o site [Grafana Dashboards](https://grafana.com/grafana/dashboards/?search=zabbix) e procure por dashboards para Zabbix.
2. Copie o ID do dashboard que você gostou.
3. No Grafana, vá para o menu **Dashboards**.
4. Clique em **New** e depois em **Import**.
5. Cole o ID do dashboard no campo "Import via grafana.com" e clique em **Load**.
6. Na próxima tela, na parte inferior, certifique-se de selecionar o seu **Data Source** do Zabbix (que você criou no passo 2.2.4) e clique em **Import**.

Exemplo de dashboard:
- [Zabbix - Full Server Status](https://grafana.com/grafana/dashboards/5363-zabbix-full-server-status/)
- [Zabbix Server Dashboard (para monitorar o Zabbix Server em si)](https://grafana.com/grafana/dashboards/8955-zabbix34-server/)

> [!WARNING]
> Alguns dashboards podem exigir permissões adicionais ou configurações específicas no Zabbix. Certifique-se de ler a descrição do dashboard e verificar se você tem os dados necessários configurados no Zabbix.


Pronto! Agora você tem o Grafana instalado e integrado com o Zabbix. Você terá um dashboard completo com dados vindos diretamente do seu Zabbix. A partir daí, você pode explorar a criação de seus próprios painéis e gráficos personalizados.

