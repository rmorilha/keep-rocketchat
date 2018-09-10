# keep-rocketchat
Repository destinated to docker-compose file for RocketChat

## Pré-requisitos:
1. Ter o yml do docker compose do RocketChat. Você consegue isso aqui no nosso github.
2. Ter os conceitos de módulos do Apache2 no Ubuntu.
3. Ter os conceitos básicos sobre proxy reverso.
4. Ter os conceitos básicos sobre SSL Challenge.
5. Compreender o uso dos arquivos de certificados SSL e redirecionamento HTTP.

## Mãos na massa !

1. Instale o Docker-CE e Docker-Compose no servidor.
**Caso seu Ubuntu (ou derivado) tenha algum docker instalado, desinstale-o**
```
sudo apt-get remove docker docker-engine docker.io
```

**Pré-Requisitos Docker**
```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

**Importação da chave GPG do repositório Docker**
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

**Improtar o repositório**
```
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

**Atualize o banco de dados do APT com o repositório recém adicionado**
```
sudo apt-get update
```

**Instale efetivamente o Docker Community Edition**
```
sudo apt-get install -y docker-ce
```

**Configure o Docker para subir junto com o sistema**
```
sudo systemctl enable docker
```

**Instale o docker-compose**
```
sudo apt-get install -y docker-compose
```


2. Instale o Apache 2
```
sudo apt-get install apache2
```


3. Instale o `CertBot/LetsEncrypt` (Necessário que você tenha um cadastro junto com o pessoal da LetsEncrypt/CertBot)
**Adicione o Repositório (PPA) do certbot**
```
sudo add-apt-repository ppa:certbot/certbot
```

**Atualize o banco de dados do APT com o repositório recém adicionado**
```
sudo apt-get update
```

**Verifique qual a versão do Python que está configurada como padrão para seu S.O.**
```
python --version
```

**Instale os pacotes do CertBot para Apache (Caso seu Python seja 2.XXX)**
```
sudo apt-get install python-certbot python-certbot-apache
```

**Instale os pacotes do CertBot para Apache (Caso seu Python seja 3.XXX)**
```
sudo apt-get install python3-certbot python3-certbot-apache
```


4. Suba os containeres do rocketchat, mongodb e mongodb-init-replica
Na pasta onde está o YML do docker-compose, execute o comando abaixo:
```
docker-compose up -d
```

**Neste docker-compose a porta do Rocketchat está configurada para `3100->3000`**
**Para saber mais sobre portas no docker, olhe este artigo: [Docker e Cerveja !](https://keep-linux.blogspot.com/2018/05/containers-docker-e-cerveja.html)**


5. Após ter os containeres rodando (`docker ps -a`), configure o Apache 2 como Reverse Proxy para os domínios (Iremos assumir aqui que você já configurou o subdomínio apontando para o servidor em seu DNS)

### Configuração Apache HTTP (porta 80).
**Crie um arquivo em `/etc/apache2/sites-available/` com o nome do seu domínio.**
```
vim /etc/apache2/sites-available/chat.keep-linux.tk.conf
```

**Insira o conteúdo abaixo:**
```
<VirtualHost *:80>
	ServerName chat.keep-linux.tk

	ServerAdmin keeplinuxbr@gmail.com

	ErrorLog ${APACHE_LOG_DIR}/chat_error.log
	CustomLog ${APACHE_LOG_DIR}/chat_access.log combined
	Redirect permanent / https://chat.keep-linux.tk/
</VirtualHost>
```

**Crie um link simbólico deste arquivo para `/etc/apache2/sites-enabled/`**
```
cd /etc/apache2/sites-enabled/ && sudo ln -s /etc/apache2/sites-available/chat.keep-linux.tk.conf chat.keep-linux.tk.conf
```

**Crie os arquivos abaixo:**
```
sudo touch /etc/letsencrypt/live/dummy.pem
```

### Configuração Apache HTTPS (porta 443).

**Crie um arquivo em /etc/apache2/sites-available/ com o nome do seu domínio-ssl.**
```
sudo vim /etc/apache2/sites-available/chat.keep-linux.tk-ssl.conf
```

**Insira o conteúdo abaixo:**
```
<VirtualHost *:443>
# Definindo o nome que o Apache Reverse Proxy irá ouvir (listenning)	
	ServerName chat.keep-linux.tk

	ServerAdmin keeplinuxbr@gmail.com
	
# Configure as opções para SSL
	SSLEngine on
	SSLProxyEngine On
# Defina onde estão os certificados SSL para seu domínio (na primeira execução do certbot, mantenha da forma abaixo)
	SSLCertificateFile /etc/letsencrypt/live/dummy.pem
	SSLCertificateKeyFile /etc/letsencrypt/live/dummy.pem
	SSLProxyCACertificateFile /etc/letsencrypt/live/dummy.pem

# Ative o Proxy Reverso do Apache
	ProxyPreserveHost On
	ProxyRequests Off
# Permita requisições de qualquer origem dentro do proxy
	<Proxy *>
	     Order deny,allow
	     Allow from all
        </Proxy>

	<Location />
	        Order allow,deny
	        Allow from all
	</Location>

# Configure o "roteamento" do proxy reverso caso for utilizada a websocket (Necessário apra o APP+ RocketChat)
	ProxyPass /websocket ws://127.0.0.1:3100/websocket
        ProxyPassMatch ^/sockjs/(.*)/websocket ws://127.0.0.1:3100/sockjs/$1/websocket
# Configure o Reverse Proxy para todos os paths para dentro do container RocketChat
	ProxyPass / http://127.0.0.1:3100/
        ProxyPassReverse / http://127.0.0.1:3100/

</VirtualHost>
```

**Crie um link simbólico deste arquivo para `/etc/apache2/sites-enabled/`**
```
cd /etc/apache2/sites-enabled/ && sudo ln -s /etc/apache2/sites-available/chat.keep-linux.tk-ssl.conf chat.keep-linux.tk-ssl.conf
```

**Ative os módulos de Proxy, proxy reverso e proxy de chamadas de websockets.**
```
sudo a2enmod proxy_http
sudo a2enmod proxy
sudo a2enmod ssl
sudo a2enmod proxy_wstunnel
sudo a2enmod rewrite
sudo systemctl restart apache2
```

6. Configurado o Apache 2 com os módulos corretos e as diretivas para Proxy Reverso, crie os certificados SSL para que seu HTTPS funcione corretamente com o CERTBOT.
```
sudo certbot certonly --apache
```

**Selecione o seu domínio no "wizzard" e confirme as opções. Feito isto, o certbot gerará um output com o caminho onde estão os certificados.**
GERALMENTE estes certificados ficam em `/etc/letsencrypt/live/chat.keep-linux.tk/` .

**Verificado o local de geração dos certificados, edite novamente o arquivo de configuração SSL de seu domínio `(/etc/apache2/sites-available/chat.keep-linux.tk.conf)` e altere os seguintes campos com o path dos certificados gerados:**

```
SSLCertificateFile /etc/letsencrypt/live/chat.keep-linux.tk/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/chat.keep-linux.tk/privkey.pem
SSLProxyCACertificateFile /etc/letsencrypt/live/chat.keep-linux.tk/fullchain.pem
```

**Execute um `reload` no Apache, para que as configurações novas entrem em vigor sem que o serviço sofra intermitência**
```
sudo systemctl reload apache2
```

Pronto !
Temos uma instância só nossa do RocketChat funcionando em Docker com Proxy Reverso no Apache 2.