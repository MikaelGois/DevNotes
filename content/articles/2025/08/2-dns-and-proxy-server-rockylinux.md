---
title: Configurando um servidor DNS e proxy transparente no Rocky Linux
type: docs
weight: 1
editURL: "https://devnotes.msglabs.site/articles/dns-and-proxy-server-rockylinux/"
next: /articles/2025/08/1-zabbix-and-grafana
---

Esse artigo aborda as configurações necessárias para implementar um servidor DNS e um servidor proxy no Rocky Linux, utilizando ferramentas como BIND e Squid.
Essa foi uma das atividades realizadas durante a matéria de Redes de Computadores II, na faculdade.
A configuração foi realizada no VirtualBox, com o seguinte cenário:
* Uma máquina que servirá como servidor, com Rocky Linux 10 instalado e duas interfaces de rede, uma configurada como bridge ou NAT e outra como rede interna (`intnet`). Interface interna: `enp0s8`, interface externa: `enp0s3`.
* Uma máquina que servirá como cliente, também com Rocky Linux 10 e somente uma interface de rede, configurada para se conectar à rede interna (`intnet`). Interface: `enp0s3`.

Endereços:
* Rede interna (`intnet`): `192.168.1.0/24`
* IP do servidor: `192.168.1.1`
* IP do cliente: `192.168.1.2`

A ideia é que a máquina cliente utilize o servidor DNS para resolver nomes de domínio e o servidor proxy para acessar a internet de forma controlada. Por isso, também é importante configurar as regras de firewall e as políticas de acesso no servidor proxy. No entanto, também irei configurar o roteamento NAT entre as zonas internas e externas do firewall.

> [!CAUTION]
> As configurações abordadas aqui funcionam em um ambiente controlado, para produção é necessário se aprofundar nas melhores práticas de segurança e configurações.

## Configuração das zonas de firewall e do roteamento NAT entre as zonas

O papel do NAT é realizar a tradução dos endereços de uma rede para outra, no nosso caso, entre a rede interna (intnet) e a rede externa (através da interface `enp0s3`). Para configurar no Rocky Linux, é necessário configurar a zona `internal` e `external` do servidor, indicando as interfaces e ativando a regra de NAT (`masquerade`) na interface externa. Além disso, também é necessário ativar o encaminhamento de pacotes no kernel linux adicionando o parâmetro `net.ipv4.ip_forward = 1` no arquivo `sysctl`.

### 1 - Ativando o encaminhamento no kernel

#### 1.1 - Criando o arquivo de configuração:
Digite o comando abaixo para criar o arquivo de configuração e escrever a regra de encaminhamento de pacotes:
```bash
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
```

#### 1.2 - Aplicando as configurações:
Digite o comando abaixo para aplicar as configurações:
```bash
sudo sysctl -p /etc/sysctl.d/99-ip-forward.conf
```

### 2 - Configurando a zona interna e externa no firewall

No nosso exemplo, usaremos a interface `enp0s8` como a zona interna e a interface `enp0s3` como a zona externa.

Você pode configurar os IPs usando o utilitário `nmcli` do NetworkManager ou a sua versão com interface gráfica `nmtui`. Após as mudanças, desative e ative as interfaces novamente para que as novas configurações tenham efeito.

Você pode verificar as configurações de IP com o comando `ip a` e as rotas com o comando `ip route`.

#### 2.1 - Alterando as interfaces das zonas internas e externas:
Digite o comando abaixo para alterar a interface da zona interna:
```bash
sudo firewall-cmd --zone=internal --change-interface=enp0s8 --permanent
```

#### 2.2 - Alterando as interfaces das zonas externas:
Digite o comando abaixo para alterar a interface da zona externa:
```bash
sudo firewall-cmd --zone=external --change-interface=enp0s3 --permanent
```

### 3 - Habilitando o NAT na interface externa
O `masquerade` permite que os pacotes da rede interna sejam traduzidos para a interface externa. Ele já vem configurado por padrão na zona externa, mas se quiser garantir que está ativado, use o comando:
```bash
sudo firewall-cmd --zone=external --add-masquerade --permanent
```

Caso ele já esteja ativado, você verá uma mensagem informando que a regra já existe.

### 4 - Configurando a política de roteamento

A política de roteamento permite controlar o tráfego entre as zonas. Além disso, resolve problemas comuns que possam impedir a comunicação e filtragem de pacotes que são desejáveis.

#### 4.1 - Criando a política de roteamento:
Para criar uma nova política de roteamento entre a zona interna e a zona externa, use o comando:
```bash
sudo firewall-cmd --new-policy=int-to-ext --permanent
```

#### 4.2 - Definindo as zonas de entrada e saída:
Para definir a zona de entrada (`ingress`), use o comando abaixo:
```bash
sudo firewall-cmd --policy=int-to-ext --add-ingress-zone=internal --permanent
```

Já para definir a zona de saída (`egress`), use o comando abaixo:
```bash
sudo firewall-cmd --policy=int-to-ext --add-egress-zone=external --permanent
```

#### 4.3 - Definindo a zona padrão:
Também é importante configurar a zona internal como a zona padrão do servidor:
```bash
sudo firewall-cmd --set-default-zone=internal
```

#### 4.4 - Definindo a política de roteamento padrão:
Para definir a política de roteamento padrão, use o comando abaixo:
```bash
sudo firewall-cmd --policy=int-to-ext --set-target=ACCEPT --permanent
```




### 5 - Aplicando mudanças no firewall
Após realizar todas as alterações, é necessário recarregar as configurações do firewall para que as mudanças tenham efeito:
```bash
sudo firewall-cmd --reload
```

### 6 - Testando o roteamento NAT através das zonas

Para testar o roteamento NAT, vamos usar o `tcpdump` para monitorar o tráfego nas interfaces. Mas primeiro é preciso garantir que o cliente esteja com as configurações de rede corretas e esteja gerando tráfego.

#### 6.1 - Configurando o cliente:

No cliente, você pode definir o gateway padrão para o IP do servidor. Faça isso usando a ferramenta `nmtui`
```bash
sudo nmtui
```

Na interface do modo texto que abrir, configure o IP da máquina, um servidor DNS temporário e o gateway padrão para o IP do servidor.

#### 6.2 - Gerando tráfego a partir do cliente:

No cliente, você pode usar o comando `ping` para gerar tráfego ICMP. Por exemplo:
```bash
ping google.com
```

Isso vai nos permitir verificar se o tráfego está sendo roteado corretamente.

#### 6.3 - Monitorando o tráfego na interface interna:
Para escutar as requisições icmp na interface interna, use o comando a seguir:
```bash
sudo tcpdump -ni enp0s8 icmp
```

Se estiverem chegando requisições ICMP na interface interna a partir do cliente, você verá as requisições sendo exibidas no terminal. Isso significa que o tráfego do cliente está chegando à interface interna do servidor.

Caso não esteja chegando nada, verifique as configurações de rede do cliente e do servidor.

#### 6.4 - Monitorando o tráfego na interface externa:
Para escutar requisições icmp na interface externa, em outra janela do terminal, use o comando abaixo:
```bash
sudo tcpdump -ni enp0s3 icmp
```

Se estiverem chegando requisições ICMP na interface externa a partir da interface interna, você verá as requisições sendo exibidas no terminal. Isso significa que o tráfego do cliente está sendo roteado corretamente através das zonas interna e externa do servidor.

Caso não esteja chegando nada, verifique as regras de roteamento e as configurações de firewall.




## Configuração do Servidor DNS

Para configurar o servidor DNS, utilizaremos o BIND (Berkeley Internet Name Domain), que é um dos servidores DNS mais populares e amplamente utilizados.

> [!TIP]
> Para explicações mais detalhadas sobre cada passo e cada parâmetro, visite [o meu artigo no GitHub](https://github.com/MikaelGois/setting-bind-dns/), e é claro, a [documentação do BIND](https://bind9.readthedocs.io/en/latest/).
> Se desejar apenas os comandos e exemplos diretos de configuração do BIND tanto para SO RHEL like quanto para SO Debian like, você pode encontrá-los [neste link](https://mikaelgois.github.io/setting-bind-dns/).

### 1 - Instalando o BIND no servidor

Para instalar o BIND e suas ferramentas utilitárias, execute o seguinte comando:
```bash
sudo dnf install bind bind-utils -y
```




### 2 - Alterando as configurações do BIND
Abra o arquivo de configuração do BIND em `/etc/named.conf`. Você pode editá-lo com o seu editor de texto favorito. Aqui usaremos o `vi`:
```bash
sudo vi /etc/named.conf
```

Digite `i` para entrar no modo de inserção e faça as alterações necessárias.

#### 2.1 - Realizando configurações básicas:
Adicione ou modifique as seguintes linhas no arquivo `named.conf` para configurar o BIND:
```bash {filename="named.conf",linenos=table}
acl "rede_interna" {
    127.0.0.1;
    192.168.1.0/24;
};

options {
    listen-on port 53 { 127.0.0.1; 192.168.1.1; };
    listen-on-v6 port 53 { ::1; };

    directory "/var/named";
    dump-file "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    secroots-file "/var/named/data/named.secroots";
    recursing-file "/var/named/data/named.recursing";

    allow-transfer { none; };
    allow-query { rede_interna; };

    forwarders {
        8.8.8.8; // Google Public DNS
        1.1.1.1; // Cloudflare DNS
        8.8.4.4; // Google Public DNS
        1.0.0.1; // Cloudflare DNS
    };
    forward first;

    recursion yes;
    allow-recursion { rede_interna; };

    dnssec-validation auto;

    managed-keys-directory "/var/named/dynamic";
    geoip-directory "/usr/share/GeoIP";

    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";

    /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
    include "/etc/crypto-policies/back-ends/bind.config";
};
```

| Principais Parâmetros     | Função                                                                                    |
|-------------------------- | ----------------------------------------------------------------------------------------- |
| `acl "rede_interna"`      | Define uma lista de controle de acesso para a rede interna.                               |
| `options`                 | Configurações gerais do servidor DNS, como portas de escuta e diretórios de arquivos.     |
| `listen-on port 53`       | Define em quais endereços IP o servidor DNS escutará as consultas.                        |
| `listen-on-v6 port 53`    | Define em quais endereços IPv6 o servidor DNS escutará as consultas.                      |
| `directory`               | Diretório onde os arquivos de zona estão localizados.                                     |
| `allow-transfer`          | Controla quais servidores podem transferir zonas do servidor DNS.                         |
| `allow-query`             | Define quem pode fazer consultas DNS.                                                     |
| `forwarders`              | Lista de servidores DNS para os quais as consultas não resolvidas devem ser encaminhadas. |
| `recursion`               | Habilita ou desabilita a recursão para consultas DNS.                                     |
| `allow-recursion`         | Define quais redes podem fazer consultas recursivas.                                      |
| `dnssec-validation`       | Habilita a validação DNSSEC.                                                              |
| `managed-keys-directory`  | Diretório para chaves gerenciadas.                                                        |
| `geoip-directory`         | Diretório para arquivos GeoIP.                                                            |
| `pid-file`                | Arquivo PID do processo do servidor DNS.                                                  |
| `session-keyfile`         | Arquivo de chave de sessão para o servidor DNS.                                           |
| `include`                 | Inclui configurações adicionais do sistema.                                               |

Pressione `Esc` e digite `:wq` para salvar e sair do editor.

#### 2.2 - Configurando logs:

Os logs do BIND são configurados na seção `logging` do arquivo de configuração, e são importantes para monitorar a atividade do servidor DNS. Aqui está um exemplo de configuração de logs no arquivo `named.conf`:

```bash {filename="named.conf",linenos=table,linenostart=42}
logging {
    channel default_log {
        file "data/named.log" versions 3 size 20m;
        severity dynamic;
        print-time yes;
        print-category yes;
        print-severity yes;
    };
    category default {
        default_log;
    };
};
```
| Principais Parâmetros | Função                                                |
| --------------------- | ----------------------------------------------------- |
| `logging`             | Configura o sistema de logs do BIND.                  |
| `channel`             | Define um canal de log.                               |
| `file`                | Especifica o arquivo de log.                          |
| `severity`            | Define o nível de severidade dos logs.                |
| `print-time`          | Habilita a impressão do timestamp nos logs.           |
| `print-category`      | Habilita a impressão da categoria nos logs.           |
| `print-severity`      | Habilita a impressão da severidade nos logs.          |
| `category`            | Define quais categorias de logs serão registradas.    |

Pressione `Esc` e digite `:wq` para salvar e sair do editor.

#### 2.3 - Configurando inclusão de arquivos:

O BIND permite a inclusão de arquivos de configuração adicionais usando a diretiva `include`. Isso é útil para organizar a configuração em vários arquivos. No final do arquivo `named.conf` adicione as seguintes linhas caso não existam:

```bash {filename="named.conf",linenos=table,linenostart=58}
include "/etc/rndc.key";
controls {
        inet 192.168.1.1 allow {
        localhost;
        192.168.1.1; }
        keys { rndc-key; };
};

include "/etc/named.main.local.zones"; // Zona de pesquisa direta e reversa que será criada
include "/etc/named.rfc1912.zones"; // Zonas de pesquisa direta e reversa padrão
include "/etc/named.root.key"; // Chave de autenticação para o servidor raiz
```

| Principais Parâmetros | Função                                                                                                                                |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `include`             | Inclui arquivos de configuração adicionais.                                                                                           |
| `controls`            | Define as permissões de controle para o servidor DNS. Está sendo adicionado aqui para configurar as permissões de controle do `rndc`. |

Pressione `Esc` e digite `:wq` para salvar e sair do editor.




### 3 - Configurando zonas de DNS

#### 3.1 - Configurando Root Hints:
Os Root Hints são informações sobre os servidores DNS raiz e são essenciais para a resolução de nomes. O BIND já vem com um arquivo de Root Hints chamado `named.ca`, que contém essas informações. O arquivo `named.conf` deve incluir a configuração abaixo, de preferência antes dos `include` para fins de organização:

```bash {filename="named.conf",linenos=table,linenostart=54}
zone "." IN {
        type hint;
        file "named.ca";
};
```

| Parâmetros    | Função                                                |
| ------------- | ----------------------------------------------------- |
| `zone`        | Define uma zona DNS.                                  |
| `type`        | Especifica o tipo de zona (ex: master, slave, hint).  |
| `file`        | Indica o arquivo de zona correspondente.              |

#### 3.2 - Configurando zonas de consulta direta e reversa:

As zonas de consulta direta são usadas para resolver nomes de domínio em endereços IP, já as zonas de consulta reversa são usadas para resolver endereços IP em nomes de domínio. Para configurar uma zona direta e reversa, você precisa criar um arquivo de zona e adicionar as entradas necessárias.
```bash
sudo vi /etc/named.main.local.zones
```

Digite `i` para entrar no modo de inserção e faça as alterações necessárias.

Dentro do arquivo, adicione as informações das zonas:
```bash {filename="named.main.local.zones",linenos=table}
zone "main.local" IN {
    type master;
    file "main.local.zone";
    allow-update {
        key rndc-key;
    };
};

zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "1.168.192.zone";
    allow-update {
        key rndc-key;
    };
};
```

| Parâmetros        | Função                                                                                                                                                                            |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `zone`            | Define uma zona DNS.                                                                                                                                                              |
| `type`            | Especifica o tipo de zona (ex: master, slave, hint). O tipo `master` indica que este servidor detém a cópia autoritativa e editável dos dados da zona.                            |
| `file`            | Especifica o nome do arquivo que contém os registros da zona. Este caminho é relativo ao diretório especificado na diretiva directory no bloco options (geralmente /var/named/).  |
| `allow-update`    | Define as permissões de atualização da zona, sobrescrevendo as configurações padrão em `named.conf`.                                                                              |

> [!NOTE]
> Os nomes de zona reversa são baseados em endereços IP e são escritos de forma invertida com a adição de `.in-addr.arpa`. Por exemplo, a zona reversa para a rede `192.168.1.0/24` seria `1.168.192.in-addr.arpa`.

Pressione `Esc` e digite `:wq` para salvar e sair do editor.

#### 3.3 - Conceda a propriedade e as permissões adequadas ao arquivo de declaração de zona:
Para garantir que o BIND tenha acesso ao arquivo de zonas, você deve definir a propriedade e as permissões corretas:
```bash
sudo chown root:named /etc/named/named.main.local.zones
sudo chmod 644 /etc/named/named.main.local.zones
```




### 4 - Criando registros de zonas de consulta direta e reversa

#### 4.1 - Configuração do arquivo de registros de zona direta:
Crie um arquivo com os registros de zona direta em `/var/named/` com o nome do seu domínio, no caso desse exemplo, `main.local.zone`:
```bash
sudo vi /var/named/main.local.zone
```

Digite `i` para entrar no modo de inserção e faça as alterações necessárias.

Adicione as seguintes configurações ao arquivo de registros de zona direta:
``` bash {filename="main.local.zone",linenos=table}
$TTL 1D  
@       IN      SOA     dns1.main.local. admin.main.local. (  
                        2025082201      ; Serial (Formato YYYYMMDDNN)
                        1H              ; Refresh
                        15M             ; Retry
                        1W              ; Expire
                        1D )            ; Negative Cache TTL

@       IN      NS      dns1.main.local.
@       IN      NS      192.168.1.1
dns1    IN      A       192.168.1.1  
www     IN      CNAME   dns1.main.local.
cliente IN      A       192.168.1.2
```

| Parâmetros    | Função                                                    |
| ------------- | --------------------------------------------------------- |
| `$TTL`        | Define o `Time To Live` padrão para os registros da zona. |
| `@`           | Representa o nome da zona atual.                          |
| `IN`          | Indica que o registro é do tipo Internet.                 |
| `SOA`         | Define o registro de autoridade da zona.                  |
| `NS`          | Define os servidores de nomes autoritativos para a zona.  |
| `A`           | Define um registro de endereço IPv4.                      |
| `CNAME`       | Define um registro de nome canônico.                      |

Pressione `Esc` e digite `:wq` para salvar e sair do editor.

#### 4.2 - Configuração do arquivo de registros de zona reversa:

Crie um arquivo com os registros de zona reversa em `/var/named/` com o nome do seu domínio, no caso desse exemplo, `1.168.192.zone`:
```bash
sudo vi /var/named/1.168.192.zone
```

Digite `i` para entrar no modo de inserção e faça as alterações necessárias.

Adicione as seguintes configurações ao arquivo de registros de zona reversa:
```bash {filename="1.168.192.zone",linenos=table}
$TTL 1D
@       IN      SOA     dns1.lab.local. admin.lab.local. ( 
                        2025082201      ; Serial (Formato YYYYMMDDNN)
                        1H              ; Refresh
                        15M             ; Retry
                        1W              ; Expire
                        1D )            ; Negative Cache TTL
@       IN      NS      dns1.lab.local.
1       IN      PTR     dns1.main.local.
2       IN      PTR     cliente.main.local.
```

| Parâmetros    | Função                                                    |
| ------------- | --------------------------------------------------------- |
| `$TTL`        | Define o `Time To Live` padrão para os registros da zona. |
| `@`           | Representa o nome da zona atual.                          |
| `IN`          | Indica que o registro é do tipo Internet.                 |
| `SOA`         | Define o registro de autoridade da zona.                  |
| `NS`          | Define os servidores de nomes autoritativos para a zona.  |
| `PTR`         | Define um registro de ponteiro para resolução reversa.    |

> [!NOTE]
> Note que os registros PTR contém apenas o último octeto do endereço IP. Esse tipo de registro é usado para a resolução reversa de nomes, permitindo que um endereço IP seja mapeado para um nome de domínio.

Pressione `Esc` e digite `:wq` para salvar e sair do editor.

#### 4.3 - Conceda a propriedade e as permissões adequadas aos arquivos de registros de zona:
Para garantir que o BIND tenha acesso ao arquivo de zonas, você deve definir a propriedade e as permissões corretas:
```bash
sudo chown root:named /var/named/main.local.zone
sudo chown root:named /var/named/1.168.192.zone
sudo chmod 640 /var/named/main.local.zone
sudo chmod 640 /var/named/1.168.192.zone
```




### 5 - Testando a sintaxe e aplicando as configurações

> [!WARNING]
> Os testes verificarão a sintaxe dos arquivos de configuração, mas não garantirão que o serviço do BIND inicie corretamente, pois pode haver problemas de lógica.

#### 5.1 - Teste a sintaxe do arquivo de configuração do BIND:
```bash
sudo named-checkconf
```

Se não houver erros, não haverá saídas. Caso haja erros, eles serão exibidos no terminal.

No caso de haver erros, e a mensagem de erro indicar a linha onde o erro ocorreu, você pode abrir o arquivo diretamente na linha indicada. Digamos que o erro esteja na linha 42 do arquivo de configuração do BIND:
```bash
sudo vi +42 /etc/named.conf
```
Depois de realizar a correção, salve, saia do editor e teste a configuração novamente.

#### 5.2 - Teste a sintaxe dos arquivos de zona:

```bash
sudo named-checkzone main.local /var/named/main.local.zone
sudo named-checkzone 1.168.192.in-addr.arpa /var/named/1.168.192.zone
```

Se não houver erros, a saída deverá indicar o `serial` do arquivo verificado e `OK`. Caso haja erros, eles serão exibidos no terminal.

No caso de haver erros, e a mensagem de erro indicar a linha onde o erro ocorreu, você pode abrir o arquivo diretamente na linha indicada. Digamos que o erro esteja na linha 2 do arquivo de configuração de zona direta:
```bash
sudo vi +2 /var/named/main.local.zone
```
Após realizar a correção, salve, saia do editor e teste a configuração novamente.

#### 5.3 - Inicie o serviço do BIND e habilite-o para iniciar automaticamente no boot:
Caso os testes de sintaxe tenham sido bem-sucedidos, você pode iniciar o serviço do BIND e habilitá-lo para iniciar automaticamente no boot:
```bash
sudo systemctl enable --now named
```

Caso ocorram erros na inicialização do serviço, você poderá verificar o status do serviço com o seguinte comando:
```bash
sudo systemctl status named
```

Você também pode verificar os logs do sistema para obter mais informações sobre o que pode estar causando o problema:
```bash
sudo journalctl -xe
```

Realize as correções necessárias e tente reiniciar o serviço novamente.




### 6 - Configurações de firewall

#### 6.1 - Permitindo tráfego DNS:
Para permitir o tráfego DNS (porta 53) no firewall, você pode usar o seguinte comando:
```bash
sudo firewall-cmd --zone=internal --permanent --add-service=dns
```

#### 6.2 - Aplicando as alterações:
Para aplicar as alterações feitas no firewall, execute o seguinte comando:
```bash
sudo firewall-cmd --reload
```




### 7 - Realizando testes de DNS

Os testes podem ser realizados tanto a partir do servidor BIND quanto de um cliente DNS.

#### 7.1 - Testando a resolução Direta: 
Para consultar um registro A para dns1.main.local, especificando que a consulta deve ser enviada para o nosso servidor BIND em 192.168.1.1:
```bash
dig @192.168.1.1 dns1.main.local A
```

Deverá haver um campo com `NOERROR` na resposta, além disso, a seção `ANSWER SECTION` na saída deve mostrar o endereço IP correto (192.168.1.1).

Caso apareça um campo indicando erro, verifique as suas configurações de zonas e do arquivo de configuração do BIND.

#### 7.2 - Testando a resolução Reversa: 
Para realizar uma consulta de pesquisa reversa (`PTR`) para o IP `192.168.1.1`, especificando que a consulta deve ser enviada para o nosso servidor BIND em `192.168.1.1`, usa-se a opção `-x`:
```bash
dig @192.168.1.1 -x 192.168.1.1
```

Deverá haver um campo com `NOERROR` na resposta, além disso, a seção `ANSWER SECTION` deve mostrar o nome de host correto (dns1.main.local.).

Caso apareça um campo indicando erro, verifique as suas configurações de zonas e do arquivo de configuração do BIND.

#### 7.3 - Testando a Resolução de nomes externos:
Para consultar um registro A para um nome de domínio externo, como `www.google.com`, especificando que a consulta deve ser enviada para o nosso servidor BIND em `192.168.1.1`:
```bash
dig @192.168.1.1 www.google.com A
```

Deverá haver um campo com `NOERROR` na resposta, além disso, a seção `ANSWER SECTION` na saída deve mostrar o endereço IP correto.

Caso apareça um campo indicando erro, verifique as suas configurações de zonas e do arquivo de configuração do BIND.




### 8 - Configurando cliente e verificando requisições DNS

#### 8.1 - Configurando o DNS no cliente:
Na máquina cliente, utilize o utilitário `nmtui` para configurar o DNS para apontar para o servidor BIND em `192.168.1.1`. Após isso, você pode testar a resolução de nomes usando o comando `dig` ou `nslookup` para garantir que a configuração está funcionando corretamente.

#### 8.2 - Verificando as requisições DNS no servidor:

Para verificar se as requisições DNS estão chegando ao servidor BIND, você pode usar o comando `tcpdump` para monitorar o tráfego na porta 53:

```bash
sudo tcpdump -i enp0s3 -n 'port 53 and host 192.168.1.1'
```

Isso mostrará as requisições DNS que estão sendo feitas para o servidor. Você pode usar isso em conjunto com os testes de resolução de nomes que você realizou anteriormente para garantir que tudo está funcionando corretamente.

Com isso, concluímos a configuração do servidor DNS no Rocky Linux. Agora, você pode aproveitar os benefícios de um servidor DNS local, como resolução de nomes mais rápida e controle sobre os registros DNS.




## Configuração do Servidor Proxy

Para servidor proxy, utilizaremos o Squid, que é um servidor proxy amplamente utilizado e confiável. A configuração abordada aqui é relativamente simples e rápida se comparada ao BIND.

### 1 - Instalando o Squid no servidor

#### 1.1 - Adicionando o repositório EPEL

Para instalar o Squid, primeiro é necessário adicionar o repositório EPEL:
```bash
sudo dnf install epel-release
```

#### 1.2 - Instalando o Squid

Após a adição do repositório EPEL, você pode instalar o Squid com o seguinte comando:
```bash
sudo dnf install squid
```

### 2 - Configurando o Squid

#### 2.1 - Editando o arquivo de configuração:

O arquivo de configuração principal do Squid é o `/etc/squid/squid.conf`. Você pode editá-lo com o seu editor de texto favorito, aqui usaremos o `vi`:
```bash
sudo vi /etc/squid/squid.conf
```

Para entrar no modo de edição, pressione `i`. Após fazer as alterações necessárias, pressione `Esc` e digite `:wq` para salvar e sair.

#### 2.2 - Configurando as ACLs:

As ACLs (Access Control Lists) são usadas pelo Squid para controlar o acesso ao proxy. Você pode definir ACLs no arquivo de configuração. Aqui está um exemplo básico:

```bash {filename="squid.conf",linenos=table}
acl localnet_v4 src 192.168.1.0/24
acl localnet_v6 src fd00::/8

acl SSL_ports port 443
acl Safe_ports port 80          # http
acl Safe_ports port 21          # ftp
acl Safe_ports port 443         # https
acl Safe_ports port 70          # gopher
acl Safe_ports port 210         # wais
acl Safe_ports port 1025-65535  # unregistered ports
acl Safe_ports port 280         # http-mgmt
acl Safe_ports port 488         # gss-http
acl Safe_ports port 591         # filemaker
acl Safe_ports port 777         # multiling http
```

| Parâmetros        | Função                                                    |
| ----------------- | --------------------------------------------------------- |
| `acl localnet`    | Define uma lista de controle de acesso.                   |
| `acl SSL_ports`   | Define uma lista de portas seguras para conexões SSL.     |
| `acl Safe_ports`  | Define uma lista de portas seguras para conexões HTTP.    |

Pressione `Esc` e digite `:wq` para salvar e sair.

#### 2.3 - Configurando as regras de acesso:

As regras de acesso controlam quem pode usar o proxy. Aqui está um exemplo básico de regras de acesso:

```bash {filename="squid.conf",linenos=table,linenostart=16}
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports

http_access allow localhost
http_access allow localnet_v4
http_access allow localnet_v6

http_access deny manager
http_access deny to_localhost
http_access deny to_linklocal
http_access deny all

http_port 3128 intercept
http_port 127.0.0.1:3129
```

| Parâmetros                    | Função                                                                                                                                                            |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `http_access deny`            | Nega o acesso a determinadas fontes ou destinos.                                                                                                                  |
| `http_access allow`           | Permite o acesso a determinadas fontes ou destinos.                                                                                                               |
| `http_port 3128 intercept`    | Define as portas em que o proxy escuta. O `3128 intercept` só faz sentido se você estiver usando transparente (interceptação) no firewall, o qual é o nosso caso. |
| `http_port 127.0.0.1:3129`    | Define uma porta adicional em que o proxy escuta. Foi configurada para resolver o problema de `no forward-proxy port configured` ao reiniciar o serviço.          |

Pressione `Esc` e digite `:wq` para salvar e sair.

#### 2.4 - Outras configurações:

```bash {filename="squid.conf",linenos=table,linenostart=31}
coredump_dir /var/spool/squid

refresh_pattern ^ftp:           1440    20%     10080
refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
refresh_pattern .               0       20%     4320
```

| Parâmetros        | Função                                                                    |
| ----------------- | ------------------------------------------------------------------------- |
| `coredump_dir`    | Define o diretório onde os arquivos de despejo de núcleo são armazenados. |
| `refresh_pattern` | Define padrões de atualização para diferentes tipos de conteúdo.          |

Pressione `Esc` e digite `:wq` para salvar e sair.




### 3 - Aplicando as configurações

Após editar o arquivo de configuração do Squid, é necessário reiniciar o serviço para que as alterações tenham efeito. Também é importante garantir que o serviço esteja habilitado para iniciar automaticamente com o sistema. Você pode fazer isso com o seguinte comando:

```bash
sudo systemctl enable --now squid
```




### 4 - Configurações de firewall

Para garantir que o Squid funcione corretamente, você pode precisar ajustar as configurações do firewall.

#### 4.1 - Adicionando o serviço do Squid no firewall:
Para liberar o tráfego do Squid, você pode usar o seguinte comando:
```bash
sudo firewall-cmd --permanent --add-service=squid
```

#### 4.2 - Redirecionando tráfego http para o Squid:
Para redirecionar o tráfego HTTP para o Squid, você pode usar as seguintes regras no `firewalld`:
```bash
sudo firewall-cmd --zone=internal --add-forward-port=port=80:proto=tcp:toport=3128 --permanent
```

#### 4.3 - Aplicando as regras de firewall:
Após adicionar as regras, não se esqueça de recarregar o `firewalld` para que as alterações tenham efeito:
```bash
sudo firewall-cmd --reload
```




### 5 - Testando a configuração

Após concluir todas as etapas de configuração, é importante testar se o Squid está funcionando corretamente. Para isso, iremos usar o navegador modo texto `elinks` na máquina cliente, e acompanhar o log de acesso do Squid no servidor.

#### 5.1 - Instalando o elinks no cliente:
Para instalar o `elinks`, é necessário ter adicionado o repositório EPEL. Você pode fazer isso com o seguinte comando:
```bash
sudo dnf install epel-release
```
Em seguida, instale o `elinks`:
```bash
sudo dnf install elinks
```

#### 5.2 - Acessando um endereço com o elinks:

Para acessar um site usando o `elinks`, por exemplo, a página de DNS na Wikipédia, você pode usar o seguinte comando:
```bash
elinks pt.wikipedia.org/wiki/dns
```

#### 5.3 - Acompanhando o log de acesso do Squid:

No servidor, você pode acompanhar o log de acesso do Squid no servidor usando o seguinte comando:
```bash
sudo tail -f /var/log/squid/access.log
```

Se você conseguir acessar a página no cliente e visualizar as requisições no log do Squid, isso indica que o proxy está funcionando corretamente. Caso contrário, você pode precisar verificar as configurações do Squid e do firewall para garantir que tudo esteja configurado corretamente.

Com isso, concluímos a configuração do Squid como um proxy transparente no Rocky Linux. Agora, você pode aproveitar os benefícios de um proxy, como controle de acesso, cache de conteúdo e monitoramento de tráfego.