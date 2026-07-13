# GenieACS 1.2.16 no Debian 11 com MongoDB 4.4

Guia de instalação do GenieACS em ambiente Debian 11, voltado
principalmente para servidores ou máquinas virtuais cuja CPU não possui
suporte à instrução AVX.

> \*\*Aviso importante\*\*
>
> Este guia documenta um cenário de laboratório/legado validado durante
> uma implantação real. O MongoDB 5.0 ou superior exige AVX em x86\_64. O
> MongoDB 4.4 é uma versão antiga e não deve ser adotado como primeira
> opção em novos ambientes de produção. Sempre que possível, utilize
> hardware com AVX e versões atualmente suportadas do MongoDB.
>
> A seção Huawei MA5608T é um exemplo de arquitetura e provisionamento.
> A sintaxe disponível pode variar conforme a release da OLT e o
> modelo/firmware da ONT.

## Cenário utilizado

Componente   Versão

\---

Debian       11 Bullseye
MongoDB      4.4.31
Node.js      20
GenieACS     1.2.16
Nginx        Proxy reverso

## Por que MongoDB 4.4?

Durante a implantação, versões mais recentes do MongoDB encerravam com
`Illegal instruction`.

Exemplo:

``` text
Illegal instruction
status=4/ILL
```

A CPU do servidor/VM não expunha AVX. Como o MongoDB 5.0 ou superior
exige AVX em plataformas x86\_64, foi utilizado MongoDB 4.4.31 para este
cenário específico.

Verifique as flags da CPU:

``` bash
lscpu
grep -m1 '^flags' /proc/cpuinfo
```

Teste especificamente AVX:

``` bash
grep -m1 -o '\\bavx\\b' /proc/cpuinfo
```

Se não houver saída, AVX não está exposto ao sistema operacional.

Em ambientes virtualizados, valide também o tipo de CPU configurado no
hypervisor.

\---

# 1\. Preparar o Debian 11

Edite os repositórios:

``` bash
nano /etc/apt/sources.list
```

Exemplo:

``` text
deb http://deb.debian.org/debian bullseye main contrib non-free
deb-src http://deb.debian.org/debian bullseye main contrib non-free

deb http://security.debian.org/debian-security bullseye-security main contrib non-free
deb-src http://security.debian.org/debian-security bullseye-security main contrib non-free

deb http://deb.debian.org/debian bullseye-updates main contrib non-free
deb-src http://deb.debian.org/debian bullseye-updates main contrib non-free
```

Comente entradas `cdrom:` caso não utilize a mídia de instalação como
repositório.

Atualize o sistema:

``` bash
apt update
apt upgrade -y
```

Instale utilitários básicos:

``` bash
apt install -y curl gnupg ca-certificates wget
```

Opcionalmente, instale os pacotes de firmware necessários ao hardware:

``` bash
apt install -y firmware-linux firmware-linux-free firmware-linux-nonfree
```

\---

# 2\. Instalar o MongoDB 4.4.31

> O MongoDB 4.4 é utilizado aqui por compatibilidade com CPU sem AVX.
> Este é um cenário legado.

Importe a chave do repositório MongoDB 4.4:

``` bash
wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | apt-key add -
```

Adicione o repositório:

``` bash
echo "deb \[ arch=amd64 ] https://repo.mongodb.org/apt/debian bullseye/mongodb-org/4.4 main" \\
> /etc/apt/sources.list.d/mongodb-org-4.4.list
```

Atualize os índices:

``` bash
apt update
```

Veja as versões disponíveis:

``` bash
apt-cache madison mongodb-org
```

Instale a versão 4.4.31:

``` bash
apt install -y \\
mongodb-org=4.4.31 \\
mongodb-org-server=4.4.31 \\
mongodb-org-shell=4.4.31 \\
mongodb-org-mongos=4.4.31 \\
mongodb-org-tools=4.4.31
```

Evite atualização automática para outra major version:

``` bash
apt-mark hold \\
mongodb-org \\
mongodb-org-server \\
mongodb-org-shell \\
mongodb-org-mongos \\
mongodb-org-tools
```

Verifique a versão:

``` bash
mongod --version
```

Habilite e inicie o serviço:

``` bash
systemctl enable mongod
systemctl start mongod
systemctl status mongod --no-pager
```

Confirme a porta padrão:

``` bash
ss -ltnp | grep 27017
```

\---

# 3\. Instalar o Node.js 20

Configure o repositório NodeSource:

``` bash
curl -fsSL https://deb.nodesource.com/setup\_20.x | bash -
```

Instale o Node.js:

``` bash
apt install -y nodejs
```

Verifique:

``` bash
node -v
npm -v
```

Não é necessário instalar o Node.js novamente usando `setup\_lts.x`.

\---

# 4\. Instalar o GenieACS 1.2.16

Instale via NPM:

``` bash
npm install -g genieacs@1.2.16
```

Localize os binários:

``` bash
command -v genieacs-cwmp
command -v genieacs-nbi
command -v genieacs-fs
command -v genieacs-ui
```

Também é possível conferir o pacote instalado:

``` bash
npm list -g --depth=0 | grep genieacs
```

## Links simbólicos

Crie links somente se `command -v` não retornar os binários em
`/usr/bin`.

Confirme primeiro o diretório global do NPM:

``` bash
npm root -g
```

No ambiente deste laboratório, o caminho utilizado foi:

``` bash
/usr/lib/node\_modules/genieacs/bin/
```

Crie os links de forma idempotente:

``` bash
ln -sf /usr/lib/node\_modules/genieacs/bin/genieacs-cwmp /usr/bin/genieacs-cwmp
ln -sf /usr/lib/node\_modules/genieacs/bin/genieacs-nbi /usr/bin/genieacs-nbi
ln -sf /usr/lib/node\_modules/genieacs/bin/genieacs-fs /usr/bin/genieacs-fs
ln -sf /usr/lib/node\_modules/genieacs/bin/genieacs-ui /usr/bin/genieacs-ui
ln -sf /usr/lib/node\_modules/genieacs/bin/genieacs-ext /usr/bin/genieacs-ext
```

\---

# 5\. Criar usuário e diretórios

Crie um usuário dedicado:

``` bash
useradd --system --no-create-home --user-group --shell /usr/sbin/nologin genieacs
```

Crie os diretórios:

``` bash
mkdir -p /opt/genieacs/ext
mkdir -p /var/log/genieacs
```

Ajuste as permissões:

``` bash
chown -R genieacs:genieacs /opt/genieacs
chown -R genieacs:genieacs /var/log/genieacs
```

\---

# 6\. Configurar o arquivo de ambiente

Crie o arquivo:

``` bash
nano /opt/genieacs/genieacs.env
```

Adicione:

``` ini
GENIEACS\_CWMP\_ACCESS\_LOG\_FILE=/var/log/genieacs/cwmp-access.log
GENIEACS\_NBI\_ACCESS\_LOG\_FILE=/var/log/genieacs/nbi-access.log
GENIEACS\_FS\_ACCESS\_LOG\_FILE=/var/log/genieacs/fs-access.log
GENIEACS\_UI\_ACCESS\_LOG\_FILE=/var/log/genieacs/ui-access.log

GENIEACS\_DEBUG\_FILE=/var/log/genieacs/debug.yaml
NODE\_OPTIONS=--enable-source-maps

GENIEACS\_EXT\_DIR=/opt/genieacs/ext

GENIEACS\_MONGODB\_CONNECTION\_URL=mongodb://127.0.0.1/genieacs

GENIEACS\_CWMP\_WORKER\_PROCESSES=2
GENIEACS\_NBI\_WORKER\_PROCESSES=2
GENIEACS\_FS\_WORKER\_PROCESSES=2
GENIEACS\_UI\_WORKER\_PROCESSES=1
```

Gere um segredo JWT seguro:

``` bash
node -e "console.log(\\"GENIEACS\_UI\_JWT\_SECRET=\\" + require('crypto').randomBytes(128).toString('hex'))" \\
>> /opt/genieacs/genieacs.env
```

> Não publique o valor real de `GENIEACS\_UI\_JWT\_SECRET` no GitHub.

Ajuste proprietário e permissão:

``` bash
chown genieacs:genieacs /opt/genieacs/genieacs.env
chmod 600 /opt/genieacs/genieacs.env
```

Confira o arquivo:

``` bash
cat /opt/genieacs/genieacs.env
```

\---

# 7\. Configurar os serviços systemd

## CWMP

Crie:

``` bash
systemctl edit --force --full genieacs-cwmp
```

Conteúdo:

``` ini
\[Unit]
Description=GenieACS CWMP
After=network.target mongod.service
Requires=mongod.service

\[Service]
User=genieacs
Group=genieacs
EnvironmentFile=/opt/genieacs/genieacs.env
ExecStart=/usr/bin/genieacs-cwmp
Restart=always
RestartSec=5

\[Install]
WantedBy=multi-user.target
```

## NBI

``` bash
systemctl edit --force --full genieacs-nbi
```

Conteúdo:

``` ini
\[Unit]
Description=GenieACS NBI
After=network.target mongod.service
Requires=mongod.service

\[Service]
User=genieacs
Group=genieacs
EnvironmentFile=/opt/genieacs/genieacs.env
ExecStart=/usr/bin/genieacs-nbi
Restart=always
RestartSec=5

\[Install]
WantedBy=multi-user.target
```

## FS

``` bash
systemctl edit --force --full genieacs-fs
```

Conteúdo:

``` ini
\[Unit]
Description=GenieACS FS
After=network.target mongod.service
Requires=mongod.service

\[Service]
User=genieacs
Group=genieacs
EnvironmentFile=/opt/genieacs/genieacs.env
ExecStart=/usr/bin/genieacs-fs
Restart=always
RestartSec=5

\[Install]
WantedBy=multi-user.target
```

## UI

``` bash
systemctl edit --force --full genieacs-ui
```

Conteúdo:

``` ini
\[Unit]
Description=GenieACS UI
After=network.target mongod.service
Requires=mongod.service

\[Service]
User=genieacs
Group=genieacs
EnvironmentFile=/opt/genieacs/genieacs.env
ExecStart=/usr/bin/genieacs-ui
Restart=always
RestartSec=5

\[Install]
WantedBy=multi-user.target
```

Recarregue o systemd:

``` bash
systemctl daemon-reload
```

Habilite os serviços:

``` bash
systemctl enable genieacs-cwmp genieacs-nbi genieacs-fs genieacs-ui
```

Inicie:

``` bash
systemctl start genieacs-cwmp genieacs-nbi genieacs-fs genieacs-ui
```

Verifique:

``` bash
systemctl status genieacs-cwmp genieacs-nbi genieacs-fs genieacs-ui --no-pager
```

Confirme as portas:

``` bash
ss -ltnp | grep -E '3000|7547|7557|7567'
```

Portas padrão utilizadas:

Serviço   Porta

\---

CWMP      7547/TCP
NBI       7557/TCP
FS        7567/TCP
UI        3000/TCP

Teste o CWMP localmente:

``` bash
curl -i http://127.0.0.1:7547
```

Uma resposta `405 Method Not Allowed` em um `GET` simples indica que
existe um serviço HTTP respondendo na porta. CWMP não é validado por um
`GET` comum.

\---

# 8\. Configurar logrotate

Crie:

``` bash
nano /etc/logrotate.d/genieacs
```

Adicione:

``` text
/var/log/genieacs/\*.log /var/log/genieacs/\*.yaml {
    daily
    rotate 30
    compress
    delaycompress
    dateext
    missingok
    notifempty
    copytruncate
}
```

Teste:

``` bash
logrotate -d /etc/logrotate.d/genieacs
```

\---

# 9\. Instalar e configurar o Nginx

Instale:

``` bash
apt update
apt install -y nginx
```

Se utilizar UFW:

``` bash
ufw allow 80/tcp
ufw allow 443/tcp
```

## Cenário A --- Nginx apenas para a interface web

Esta é a opção mais simples. A CPE acessa diretamente a porta CWMP
`7547`.

Crie:

``` bash
nano /etc/nginx/sites-available/genieacs
```

Exemplo:

``` nginx
server {
    listen 80;
    server\_name acs.seuprovedor.com.br;

    location / {
        proxy\_pass http://127.0.0.1:3000;

        proxy\_http\_version 1.1;
        proxy\_set\_header Host $host;
        proxy\_set\_header X-Forwarded-Proto $scheme;
        proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
        proxy\_set\_header X-Real-IP $remote\_addr;
    }
}
```

Neste cenário:

``` text
Interface web:
http://acs.seuprovedor.com.br

URL ACS configurada na CPE:
http://IP-DO-ACS:7547
```

Algumas CPEs utilizam ou aceitam um caminho adicional, por exemplo:

``` text
http://IP-DO-ACS:7547/tr069
```

Valide o comportamento no firmware do equipamento.

## Cenário B --- CWMP também atrás do Nginx

Exemplo:

``` nginx
server {
    listen 80;
    server\_name acs.seuprovedor.com.br;

    location / {
        proxy\_pass http://127.0.0.1:3000;

        proxy\_http\_version 1.1;
        proxy\_set\_header Host $host;
        proxy\_set\_header X-Forwarded-Proto $scheme;
        proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
        proxy\_set\_header X-Real-IP $remote\_addr;
    }

    location /cwmp {
        proxy\_pass http://127.0.0.1:7547;

        proxy\_http\_version 1.1;
        proxy\_set\_header Host $host;
        proxy\_set\_header X-Forwarded-Proto $scheme;
        proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
        proxy\_set\_header X-Real-IP $remote\_addr;
    }

    location /nbi {
        proxy\_pass http://127.0.0.1:7557;
    }

    location /files {
        proxy\_pass http://127.0.0.1:7567;
    }
}
```

Neste cenário, a URL ACS será:

``` text
http://acs.seuprovedor.com.br/cwmp
```

Ative o site:

``` bash
ln -s /etc/nginx/sites-available/genieacs /etc/nginx/sites-enabled/genieacs
```

Caso o link já exista, não é necessário recriá-lo.

Teste:

``` bash
nginx -t
```

Recarregue:

``` bash
systemctl reload nginx
```

\---

# 10\. HTTPS

Depois que o DNS estiver apontando para o servidor:

``` text
acs.seuprovedor.com.br -> IP público
```

Instale o Certbot:

``` bash
apt install -y certbot python3-certbot-nginx
```

Solicite o certificado:

``` bash
certbot --nginx
```

> Para produção, utilize TLS e um `GENIEACS\_UI\_JWT\_SECRET` exclusivo e
> seguro.

\---

# 11\. Autenticação de CPEs no GenieACS

No GenieACS:

``` text
Admin -> Config
```

## Aceitar sessões CWMP sem exigir autenticação HTTP

Configure:

``` text
cwmp.auth
```

Valor:

``` javascript
true
```

Isso controla a autenticação no sentido:

``` text
CPE -> ACS
```

> Aceitar qualquer CPE que consiga alcançar o endpoint CWMP aumenta a
> superfície de exposição. Em produção, filtre o acesso por
> rede/VLAN/firewall ou implemente autenticação adequada.

## Connection Request

O Connection Request ocorre no sentido:

``` text
GenieACS -> CPE
```

É utilizado, por exemplo, pelo botão `Summon`.

Se a CPE exige usuário e senha, configure `cwmp.connectionRequestAuth`
de acordo com as credenciais anunciadas/configuradas na CPE.

Exemplo de credencial fixa:

``` javascript
AUTH("genieacs", "SENHA\_DA\_CPE")
```

Um erro como:

``` text
Connection request error: Unexpected status code 401
```

indica falha de autenticação na requisição do ACS para a CPE.

Não confunda:

``` text
cwmp.auth                  = CPE -> ACS
cwmp.connectionRequestAuth = ACS -> CPE
```

\---

# 12\. Diagnóstico do tráfego CWMP

Verifique tráfego na porta 7547:

``` bash
tcpdump -ni any port 7547
```

Acompanhe o serviço CWMP:

``` bash
journalctl -fu genieacs-cwmp
```

Acompanhe o access log:

``` bash
tail -f /var/log/genieacs/cwmp-access.log
```

Verifique conexões TCP:

``` bash
ss -tn | grep 7547
```

No MikroTik:

``` routeros
/ip firewall connection print where dst-port=7547
```

Também é possível utilizar Torch na interface por onde o tráfego das
CPEs chega:

``` routeros
/tool torch interface=<INTERFACE> port=7547
```

\---

# 13\. Arquitetura de exemplo --- Huawei MA5608T

Cenário:

``` text
                    Core MikroTik
                         |
                  Trunk VLAN 100/115
                         |
                  Huawei MA5608T
                         |
                     GPON 0/0/0
                         |
                        ONT
                 \_\_\_\_\_\_\_\_|\_\_\_\_\_\_\_\_
                |                 |
       WAN de gerenciamento   Bridge de Internet
            VLAN 100              VLAN 115
                |                 |
              DHCP          Roteador do cliente
                |              PPPoE
             GenieACS
```

Objetivo:

``` text
VLAN 100 -> gerenciamento/TR-069
VLAN 115 -> bridge para o roteador PPPoE
```

A ONT possui uma WAN de gerenciamento separada do tráfego de Internet do
cliente.

> \*\*Importante:\*\* os comandos abaixo são exemplos para a família Huawei
> MA5600/MA5608T. Antes de aplicar em produção, utilize `?` na CLI e
> valide a sintaxe disponível na release da OLT.

\---

# 14\. Criar a VLAN de gerenciamento

No modo de configuração:

``` text
vlan 100 smart
```

A VLAN 115 deve existir previamente caso já seja utilizada para o
serviço PPPoE.

Valide:

``` text
display vlan 100
display vlan 115
```

\---

# 15\. Liberar VLANs no uplink

Identifique a placa/porta de uplink utilizada.

Exemplo conceitual:

``` text
interface giu 0/19
port trunk allow-pass vlan 100 115
quit
```

A sintaxe do uplink pode variar conforme a placa instalada.

Valide a configuração atual antes de alterar:

``` text
display current-configuration
```

\---

# 16\. Criar ou validar o DBA Profile

O `dba-profile-id` precisa existir antes de ser referenciado no Line
Profile.

Exemplo:

``` text
dba-profile add profile-id 100 profile-name "DBA-TR069" type4 max 1000000
```

Valide:

``` text
display dba-profile profile-id 100
```

A banda deve ser dimensionada conforme o projeto da rede.

\---

# 17\. Criar o Service Profile

Exemplo:

``` text
ont-srvprofile gpon profile-id 100 profile-name "SRV-TR069"
```

Dentro do perfil:

``` text
ont-port pots adaptive eth adaptive
port vlan eth 1 translation 115 user-vlan 115
commit
quit
```

Neste exemplo, a porta Ethernet da ONT entrega a VLAN de Internet
conforme a arquitetura adotada.

> O tratamento da VLAN de gerenciamento/TR-069 depende da capacidade
> OMCI e do firmware da ONT. Não configure a VLAN 100 como serviço de
> usuário na porta LAN sem entender o impacto.

\---

# 18\. Criar o Line Profile

Exemplo:

``` text
ont-lineprofile gpon profile-id 100 profile-name "LINE-TR069"
```

Dentro do perfil:

``` text
tcont 1 dba-profile-id 100

gem add 1 eth tcont 1
gem mapping 1 0 vlan 115

gem add 2 eth tcont 1
gem mapping 2 0 vlan 100

tr069-management enable

commit
quit
```

Resultado conceitual:

``` text
GEM 1 -> VLAN 115 -> Internet/PPPoE
GEM 2 -> VLAN 100 -> gerenciamento/TR-069
```

A criação de duas GEM Ports não significa que todo o tráfego da ONT será
convertido para VLAN 100. O mapeamento associa os serviços/VLANs às GEM
Ports configuradas.

\---

# 19\. Registrar a ONT

Entre na interface GPON:

``` text
interface gpon 0/0
```

Exemplo:

``` text
ont add 0 sn-auth HWTCXXXXXXXX omci ont-lineprofile-id 100 ont-srvprofile-id 100
```

A sintaxe exata do `ont add` pode variar por release.

Valide com:

``` text
ont add ?
```

\---

# 20\. Configurar a WAN de gerenciamento da ONT

A ONT precisa possuir conectividade IP para iniciar uma sessão CWMP com
o GenieACS.

No cenário deste guia:

``` text
VLAN 100
   |
Servidor DHCP
   |
ONT recebe IP de gerenciamento
   |
ONT acessa o GenieACS na porta 7547
```

Dependendo da ONT e da release da OLT, a WAN de gerenciamento pode ser
provisionada por OMCI através de comandos `ont ipconfig` ou por um
perfil WAN específico.

Exemplo conceitual:

``` text
ont ipconfig <PON> <ONT-ID> dhcp vlan 100
```

**Não copie esse comando diretamente sem validar a ajuda da CLI:**

``` text
ont ipconfig ?
```

Em algumas releases, a ordem e os parâmetros exigidos são diferentes.

\---

# 21\. Criar o perfil do servidor TR-069

No modo de configuração global, consulte primeiro:

``` text
ont tr069-server-profile add ?
```

Exemplo de perfil:

``` text
ont tr069-server-profile add profile-id 1
```

Informe a URL ACS conforme solicitado pela CLI/firmware:

``` text
http://http://IP\_DO\_SERVIDOR:7547
```

ou, quando utilizado domínio:

``` text
https://acs.seuprovedor.com.br/cwmp
```

Credenciais ACS só devem ser configuradas se a política de autenticação
do servidor exigir.

\---

# 22\. Vincular o perfil TR-069 à ONT

Na família MA5600, o comando encontrado em várias releases utiliza
`ont tr069-server-config` dentro da interface GPON.

Exemplo:

``` text
interface gpon 0/0
ont tr069-server-config 0 0 profile-id 1
quit
```

Onde, no exemplo:

``` text
0 = porta PON
0 = ONT ID
1 = profile-id do servidor TR-069
```

Valide obrigatoriamente na sua OLT:

``` text
ont tr069-server-config ?
```

Não utilize `ont tr069 bind` como comando genérico sem confirmar que ele
existe na release instalada.

\---

# 23\. URL ACS na CPE

Para acesso direto ao CWMP:

``` text
http://IP\_DO\_SERVIDOR:7547
```

ou:

``` text
http://100.98.52.60:7547
```

Se o firmware da CPE utiliza um caminho:

``` text
http://100.98.52.60:7547/tr069
```

No laboratório, uma Huawei HS8546V5 comunicou com o GenieACS utilizando
caminho `/tr069`.

Se o CWMP estiver atrás do Nginx conforme o cenário B:

``` text
http://acs.seuprovedor.com.br/cwmp
```

\---

# 24\. Validar se a ONT está gerando tráfego TR-069

No servidor GenieACS:

``` bash
tcpdump -ni any port 7547
```

Reinicie a ONT ou aguarde o `Periodic Inform`.

Acompanhe:

``` bash
journalctl -fu genieacs-cwmp
```

e:

``` bash
tail -f /var/log/genieacs/cwmp-access.log
```

Se nenhum pacote chegar ao servidor, valide:

* WAN de gerenciamento da ONT;
* endereço IP recebido;
* servidor DHCP da VLAN 100;
* gateway;
* roteamento;
* firewall;
* VLAN no uplink;
* mapeamento GEM/VLAN;
* URL ACS;
* cliente TR-069 habilitado na CPE.

\---

# 25\. Backup da Huawei MA5608T via TFTP

Salve a configuração antes do backup:

``` text
save
```

Siga a confirmação apresentada pela OLT.

Instale um servidor TFTP no Debian:

``` bash
apt install -y tftpd-hpa
```

Exemplo de `/etc/default/tftpd-hpa`:

``` text
TFTP\_USERNAME="tftp"
TFTP\_DIRECTORY="/srv/tftp"
TFTP\_ADDRESS=":69"
TFTP\_OPTIONS="--secure"
```

Crie o diretório:

``` bash
mkdir -p /srv/tftp
chown tftp:nogroup /srv/tftp
chmod 755 /srv/tftp
```

Para permitir upload de arquivos novos pela OLT, o `tftpd-hpa` pode
precisar da opção `--create`:

``` text
TFTP\_OPTIONS="--secure --create"
```

Reinicie:

``` bash
systemctl restart tftpd-hpa
systemctl status tftpd-hpa --no-pager
```

Confirme UDP 69:

``` bash
ss -lunp | grep ':69'
```

Na OLT:

``` text
backup configuration tftp IP\_DO\_SERVIDOR:7547 MA5608T-config-20260710.cfg
```

Durante o teste, monitore o servidor:

``` bash
tcpdump -ni any port 69
```

Se aparecer:

``` text
Failure cause: Failed to transfer the file
```

valide conectividade, firewall, permissão de escrita e suporte a criação
de arquivo pelo servidor TFTP.

\---

# 26\. Comandos úteis de diagnóstico

## GenieACS

``` bash
systemctl status genieacs-cwmp --no-pager
systemctl status genieacs-nbi --no-pager
systemctl status genieacs-fs --no-pager
systemctl status genieacs-ui --no-pager
```

``` bash
journalctl -u genieacs-cwmp -n 100 --no-pager
```

``` bash
ss -ltnp | grep -E '3000|7547|7557|7567'
```

## MongoDB

``` bash
systemctl status mongod --no-pager
```

``` bash
journalctl -u mongod -n 100 --no-pager
```

``` bash
mongod --version
```

## Nginx

``` bash
nginx -t
```

``` bash
systemctl status nginx --no-pager
```

## Rede

``` bash
ip addr
ip route
```

``` bash
tcpdump -ni any port 7547
```

\---

# 27\. Problemas encontrados durante a implantação

## MongoDB encerra com `Illegal instruction`

Sintoma:

``` text
status=4/ILL
Illegal instruction
```

Causa encontrada no laboratório:

``` text
CPU/VM sem AVX
```

Solução utilizada:

``` text
MongoDB 4.4.31
```

## `curl` na porta 7547 retorna 405

Exemplo:

``` text
405 Method Not Allowed
```

Isso não significa, isoladamente, falha do CWMP. Um `GET` comum não
representa uma sessão TR-069.

## CPE aparece no GenieACS, mas `Summon` retorna 401

Exemplo:

``` text
Connection request error: Unexpected status code 401
```

Verifique:

``` text
Device.ManagementServer.ConnectionRequestUsername
Device.ManagementServer.ConnectionRequestPassword
Device.ManagementServer.ConnectionRequestURL
```

E revise:

``` text
cwmp.connectionRequestAuth
```

## Alteração de parâmetro retorna CWMP 9002

Exemplo:

``` text
faultCode: 9002
faultString: Internal error
```

Verifique:

* caminho do parâmetro;
* suporte de escrita do firmware;
* instância WLAN correta;
* limitações do firmware da CPE;
* logs CWMP.

\---

# 28\. Considerações de segurança

Para produção:

* utilize HTTPS/TLS;
* gere um JWT secret exclusivo;
* não publique segredos no Git;
* limite o acesso ao CWMP por rede, VLAN ou firewall;
* proteja a interface web;
* não exponha NBI e FS publicamente sem necessidade;
* implemente política de autenticação de CPEs;
* mantenha backup do MongoDB e da OLT;
* monitore os logs do CWMP;
* planeje a migração de versões legadas do MongoDB.

\---

# Referências

* GenieACS Documentation: https://docs.genieacs.com/
* GenieACS GitHub: https://github.com/genieacs/genieacs
* MongoDB Production Notes:
https://www.mongodb.com/docs/manual/administration/production-notes/
* MongoDB Community: https://www.mongodb.com/
* NodeSource distributions:
https://github.com/nodesource/distributions

\---

## Autor

Documentação criada a partir de um laboratório de implantação e
troubleshooting de GenieACS/TR-069 em ambiente ISP.

O objetivo é registrar o processo, os erros encontrados e as decisões
técnicas tomadas durante a implantação.

<img width="1364" height="595" alt="vla" src="https://github.com/user-attachments/assets/bbfd19de-1069-4071-9ee0-e985e86c2360" />

