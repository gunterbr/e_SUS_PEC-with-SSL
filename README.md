# Certificado SSL com Let's Encrypt para o e-SUS-PEC
Nota: Certifique-se de não estar ocupando a porta 80, que será usada pelo certbot para baixar o certificado SSL.

## LINUX 🖥️
- Ubuntu 22.04.2
- OpenSSL 3.0.2 (Caso prefira, purgue a versão padrão e instale a versão 1 - isso poderá dispensar a conversão entre .pem, .jks e .p12)
- Java 8

### Habilitando o SSL

#### Encerre o serviço
```
service e-SUS-PEC stop
```

#### Instalação do snap
```
sudo snap install core
sudo snap refresh core
```

#### Instalação do certbot
```
sudo snap install --classic certbot
```

#### Baixando o certificado para o seu domínio
```
certbot certonly --standalone -d meudominio.com.br
```

#### Vá até a pasta de instalação do e-SUS
```
cd /opt/e-SUS/webserver/config
```

#### Converta o certificado .pem para .jks (Java KeyStore)
Nota: se você usar a versão 1 do OpenSSL é provável que funcione convertendo diretamente de .pem para .p12

```
openssl pkcs12 -export -in /etc/letsencrypt/live/meudominio.com.br/fullchain.pem -inkey /etc/letsencrypt/live/meudominio.com.br/privkey.pem -out /opt/e-SUS/webserver/config/esusaps.jks -name "esusaps" -passout pass:suaSenha
```

#### Converta o certificado .jks para .p12 (PKCS12)
```
keytool -importkeystore -srckeystore /opt/e-SUS/webserver/config/esusaps.jks -destkeystore /opt/e-SUS/webserver/config/esusaps.p12 -srcstoretype JKS -deststoretype PKCS12 -deststorepass suaSenha -srcstorepass suaSenha
```

#### Edite o application.properties
```
nano application.properties
```

Altere ou inclua
```
server.port=443
server.ssl.key-store-type=PKCS12
server.ssl.key-store=/opt/e-SUS/webserver/config/esusaps.p12
server.ssl.key-store-password=suaSenha
server.ssl.key-alias=esusaps
security.require-ssl=true
```

**INICIE O SERVICO DO E-SUS NOVAMENTE!!!**

### Renovação automática
#### Crie um uma tarefa para lidar com a renovação do certificado, conversão para .jks e .p12
```
nano renew_cert.sh
```

Inclua
```
#!/bin/bash

cert_file="/etc/letsencrypt/live/meudominio.com.br/fullchain.pem"
key_file="/etc/letsencrypt/live/meudominio.com.br/privkey.pem"

jks_file="/opt/e-SUS/webserver/config/esusaps.jks"
p12_file="/opt/e-SUS/webserver/config/esusaps.p12"

password="suaSenha"
alias="esusaps"

sudo certbot renew --force-renewal

openssl pkcs12 -export -in $cert_file -inkey $key_file -out $jks_file -name $alias -passout pass:$password

keytool -delete -alias $alias -keystore $p12_file -storetype PKCS12 -storepass $password
keytool -importkeystore -srckeystore $jks_file -destkeystore $p12_file -srcstoretype JKS -deststoretype PKCS12 -deststorepass $password -srcstorepass $password
```

#### Torne o script executável
```
chmod +x renew_cert.sh
```

#### Adicione uma tarefa cron para ser executada a cada 80 dias
```
crontab -e
```

Inclua
```
0 0 */80 * * /opt/e-SUS/webserver/config/renew_cert.sh >> /opt/e-SUS/webserver/config/certbot_renew.log 2>&1
```

## WINDOWS 🖥️
- Windows 10
- OpenSSL 1.1.1.2100
- Java 8

Nota: Esta explicação sobre como fazer o processo no Windows não é exaustiva. Verifique questões como o referenciamento dos diretórios e renovação do certificado.

### Preparando o certificado

1º Gere o certificado através do certbot [Aqui](#baixando-o-certificado-para-o-seu-dom%C3%ADnio)

2º Inicie PowerShell como ADMINISTRADOR

#### Instale o Chocolatey

```
Set-ExecutionPolicy AllSigned
```

```
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

#### Instale o OpenSSL

```
choco install openssl --version=1.1.1.2100
```

#### Converta o certificado .pem para .p12
Execute dentro do diretório onde o certificado foi salvo ou forneça o caminho do arquivo.

```
openssl pkcs12 -export -in fullchain.pem -inkey privkey.pem -out esusaps.p12 -name "esusaps" -passout pass:suaSenha
```

3º Configure, como nesse exemplo [Aqui](#edite-o-applicationproperties)

## Lembre-se:
* O novo certificado gerado e convertido só será carregador se reiniciar o serviço e-SUS-PEC
* A tarefa só terá sucesso se você manter a porta 80 livre para o certbot
