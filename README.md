# Certificado SSL com Let's Encrypt para o e-SUS-PEC
```
Nota: Certifique-se de não estar ocupando a porta 80, que será usada pelo certbot para baixar o certificado SSL.
```

## Habilitando o SSL
### Encerre o serviço
```
service e-SUS-PEC stop
```

### Instalação do snap
```
sudo snap install core
sudo snap refresh core
```

### Instalação do certbot
```
sudo snap install --classic certbot
```

### Baixando o certificado para o seu domínio
```
certbot certonly --standalone -d meudominio.com.br
```

### Vá até a pasta de instalação do e-SUS
```
cd /opt/e-SUS/webserver/config
```

### Converta o certificado .pem para .jks (Java KeyStore)
```
openssl pkcs12 -export -in /etc/letsencrypt/live/meudominio.com.br/fullchain.pem -inkey /etc/letsencrypt/live/meudominio.com.br/privkey.pem -out /opt/e-SUS/webserver/config/esusaps.jks -name "esusaps" -passout pass:suaSenha
```

### Converta o certificado .jks para .p12 (PKCS12)
```
keytool -importkeystore -srckeystore /opt/e-SUS/webserver/config/esusaps.jks -destkeystore /opt/e-SUS/webserver/config/esusaps.p12 -srcstoretype JKS -deststoretype PKCS12 -deststorepass suaSenha -srcstorepass suaSenha
```

### Edite o application.properties
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

## Renovação automática
### Crie um uma tarefa para lidar com a renovação do certificado, conversão para .jks e .p12
```
nano renew_cert.sh
```

Inclua
```
#!/bin/bash

cert_file="/etc/letsencrypt/live/caroebe.esushost.com.br/fullchain.pem"
key_file="/etc/letsencrypt/live/caroebe.esushost.com.br/privkey.pem"

jks_file="/opt/e-SUS/webserver/config/esusaps.jks"
p12_file="/opt/e-SUS/webserver/config/esusaps.p12"

password="Jtw1881"
alias="esusaps"

sudo certbot renew --force-renewal

openssl pkcs12 -export -in $cert_file -inkey $key_file -out $jks_file -name $alias -passout pass:$password

keytool -delete -alias $alias -keystore $p12_file -storetype PKCS12 -storepass $password
keytool -importkeystore -srckeystore $jks_file -destkeystore $p12_file -srcstoretype JKS -deststoretype PKCS12 -deststorepass $password -srcstorepass $password
```

### Torne o script executável
```
chmod +x renew_cert.sh
```

### Adicione uma tarefa cron para ser executada a cada 80 dias
```
crontab -e
```

Inclua
```
0 0 */80 * * /opt/e-SUS/webserver/config/renew_cert.sh >> /opt/e-SUS/webserver/config/certbot_renew.log 2>&1
```

## Lembre-se:
* O novo certificado gerado e convertido só será carregador se reiniciar o serviço e-SUS-PEC
* A tarefa só terá sucesso se você manter a porta 80 livre para o certbot