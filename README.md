# Traefik-acme-local-pki
## 📌 Описание

Этот репозиторий демонстрирует, как настроить Traefik для работы с step-ca в качестве центра сертификации.
Цель: получать и использовать сертификаты, подписанные собственным CA (Root + Intermediate), вместо Let’s Encrypt.

⸻

## Предистория. 
Изначальной я использовал Trefik с подключенным wildcard сертификатом, 
там где невозможно было получать сертификаты Let'cEncrypte.

Это можно включить вот таким способом:

👌 В файле trefik.yml подсключался провайдер типа file
```yml
  file:
    directory: /custom
    watch: true
```
и уже в файле  traefik/data/custom/**dynamic.yml**  

```yml
http:
  routers:
    traefik:
      rule: "Host(`traefik.domain.example.com`)"
      service: "api@internal"
      tls:
        domains:
          - main: "*.${DOMAIN}"
tls:
  certificates:
    - certFile: "/ssl/cert.pem"
      keyFile: "/ssl/privkey.pem"
```
собственно таким способом можно использовать wildcard сертификат в Trefik.
Но гораздо интереснее сделать свой центр сертификации и получать сертификаты там. 

# ⚙️  Шаги настройки

- Очевидно нужна возможность управлять DNS A записями.
- Нужен установленный docker и docker compose plugin.
- Нужны сертификаты, корневой сертификат и промежуточный сертификаты. 
> Подойдут любые инструменты, такие как easy-rsa, openssl или любой другой. 

## smallstep/step-ca

Сам smallstep/step-ca можно запустить по их отличной *[инструкции](https://hub.docker.com/r/smallstep/step-ca)* или *[вот тут](https://smallstep.com/docs/tutorials/docker-tls-certificate-authority/index.html)*

### C помощью docker-compose.yml можно вот так

```yml
services:
  step-ca:
    image: 'smallstep/step-ca:latest'
    container_name: 'acme-ca'
    hostname: 'acme-ca'
    ports:
      - 9000:9000
    dns:
      - '${DNS1}'
      - '${DNS2}'
    dns_search:
      - '${DOMAIN}'
    environment:
      - "DOCKER_STEPCA_INIT_NAME=${CA_NAME}"
      - "DOCKER_STEPCA_INIT_DNS_NAMES=${CA_DNS_NAMES}"
      - "DOCKER_STEPCA_INIT_PROVISIONER_NAME=${STEP_PROVISIONER_NAME}"
      - "DOCKER_STEPCA_INIT_PASSWORD=${CA_ENCRYPTION_PASS}"
      - "DOCKER_STEPCA_INIT_SSH=${INIT_SSH}"
    volumes:
      - './data:/home/step'
      - '/etc/localtime:/etc/localtime:ro'
```
В .env можно убрать все переменные

```yml
# Имя Центра Сертификации (например, PrivateCorp CA). 
# Будет видно во всех выданных сертификатах.

CA_NAME="Smalstep intermediate ACME CA"

# Список имён хостов/IP-адресов (через запятую), 
# от которых CA будет принимать запросы на выдачу сертификатов.

CA_DNS_NAMES=acme-ca.home.arpa

STEP_PROVISIONER_NAME=admin

# Пароль для шифрования ключей CA и провиженера по умолчанию.  
# Сгенерировать можно так: 'pwgen -N 1 -s 32'
CA_ENCRYPTION_PASS="<somegneratedpass>"

# Установите любое непустое значение, чтобы включить поддержку SSH-сертификатов
INIT_SSH=

# DNS-домены поиска
DOMAIN=home.arpa

# Основной DNS-сервер
DNS1=192.168.1.250

# Вторичный DNS-сервер
DNS2=192.168.1.240
```
Ну и остался конфигурационный фаил step-ca *[подробный пример есть в документации](https://smallstep.com/docs/step-ca/configuration/#example-configuration)*   

Мне хватило такого **ca.json** :

```json
{
    "root": "/home/step/certs/root-ca.crt",
    "federatedRoots": null,
    "crt": "/home/step/certs/intermediate-ca.crt",
    "key": "/home/step/secrets/intermediate-ca.key",
    "address": ":9000",
    "insecureAddress": "",
    "dnsNames": [
            "localhost",
            "acme-ca.home.arpa"
    ],
    "logger": {
            "format": "text"
    },
    "db": {
            "type": "badgerv2",
            "dataSource": "/home/step/db",
            "badgerFileLoadingMode": ""
    },
    "authority": {
            "provisioners": [
                    {
                            "type": "ACME",
                            "name": "acme",
                            "forceCN": true,
                            "claims": {
                                    "enableSSHCA": true,
                                    "disableRenewal": false,
                                    "allowRenewalAfterExpiry": false,
                                    "disableSmallstepExtensions": false,
                                    "maxTLSCertDuration": "2160h", # Это 90 дней. ;)
                                    "defaultTLSCertDuration": "2160h"
                            },
                            "options": {
                                    "x509": {},
                                    "ssh": {}
                            }
                    }
            ],
            "template": {},
            "backdate": "1m0s",
            "enableAdmin": true
    },
    "tls": {
            "cipherSuites": [
                    "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256",
                    "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256"
            ],
            "minVersion": 1.2,
            "maxVersion": 1.3,
            "renegotiation": false
    },
    
    "commonName": "acme-ca.home.arpa"
}

```
Соббственно содержимое проекта

```json
.
├── data
│   ├── certs
│   │   ├── intermediate-ca.crt # intermediate сертификат
│   │   └── root-ca.crt # Понятно из названия root сертификат
│   ├── config
│   │   └── ca.json  # Конфиг фаил для step-ca пример выше
│   ├── db # Директория, на котрую нужно дать права  chown -R 1000:1000  
│   └── secrets
│       ├── intermediate-ca.key # не сложно догадаться ключ
│       └── password  #  сюда нужно добавить пароль который создавался в .env
└── docker-compose.yml
```
Итак, помимо конфигурационных файлов нужно не забыть права на директории   
`chown -R 1000:1000 data`   
`chown 1000:1000 step/secrets/password`   
парооль из этой перменной CA_ENCRYPTION_PASS=   
`echo "<somegneratedpass>" > data/secrets/password`   

```bash
docker ocmpose up -d 
```
Если всё взлетело, то   
`curl https://localhost:9000/health`   
выдаст    
`{"status":"ok"}`   
а так можно посмотреть provisioners   
`curl https://acme-ca.home.arpa:9000/provisioners | jq`   

Если не особо, то 
```bash 
docker compose logs -f
```

























