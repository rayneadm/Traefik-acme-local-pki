# Traefik step-ca local acme pki
## Description

This repository demonstrates how to configure Traefik to work with step-ca as a Certificate Authority.
Goal: obtain and use certificates signed by your own CA (Root + Intermediate), instead of Let’s Encrypt.


## Background. 
Initially, I used Traefik with a connected wildcard certificate, 
in cases where it was not possible to obtain certificates from Let’s Encrypt.

This can be enabled like this:

In the trefik.yml file, a provider of type file was connected
```yml
  file:
    directory: /custom
    watch: true
```
and already in the file  traefik/data/custom/**dynamic.yml**  

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
Variables in .env:
```bash
DOMAIN=home.arpa
DOMAIN_API=traefik-api.home.arpa
```
Actually, this way you can use a wildcard certificate in Traefik.
But it is much more interesting to make your own Certificate Authority and obtain certificates there. And more about this below.



# Setup steps

- Obviously, you need the ability to manage DNS A records.
- Docker and docker compose plugin are required.
- Root and intermediate certificates are required. 
> Any tools for creating a certificate will work, easy-rsa, openssl, etc.. 

## smallstep/step-ca

The smallstep/step-ca itself can be run according to their excellent *[instructions](https://hub.docker.com/r/smallstep/step-ca)* or *[here](https://smallstep.com/docs/tutorials/docker-tls-certificate-authority/index.html)*

### Using docker-compose.yml like this

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
You can remove all variables in .env

```yml
# Name of Certificate Authority (for example, PrivateCorp CA). 
# Will be visible on all issued certificates.

CA_NAME="Smalstep intermediate ACME CA"

# List of hostnames/IP addresses (comma-separated), 
# from which the CA will accept requests for issuing certificates.

CA_DNS_NAMES=acme-ca.home.arpa

STEP_PROVISIONER_NAME=admin

# Password for encrypting CA keys and the default provisioner.  
# Generate like this: 'pwgen -N 1 -s 32'
CA_ENCRYPTION_PASS="<somegneratedpass>"

# Set any non-empty value to enable SSH certificate support
INIT_SSH=

# DNS search domains
DOMAIN=home.arpa

# Primary DNS server
DNS1=192.168.1.250

# Secondary DNS server
DNS2=192.168.1.240
```
And there is still the step-ca configuration file *[a detailed example is in the documentation](https://smallstep.com/docs/step-ca/configuration/#example-configuration)*   

I was satisfied with such **ca.json** :

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
                                    "maxTLSCertDuration": "2160h", # This is 90 days ;)
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
Project content

```bash
.
├── data
│   ├── certs
│   │   ├── intermediate-ca.crt # intermediate certificate
│   │   └── root-ca.crt # as the name suggests, root certificate
│   ├── config
│   │   └── ca.json  # Config file for step-ca, example above
│   ├── db # Directory that needs chown -R 1000:1000  
│   └── secrets
│       ├── intermediate-ca.key # obviously the key
│       └── password  # here you need to add the password created in .env
└── docker-compose.yml
```
So, in addition to configuration files, don’t forget permissions for directories   
```bash
chown -R 1000:1000 data   
chown 1000:1000 step/secrets/password   
#  password from this variable CA_ENCRYPTION_PASS=   
echo "<somegneratedpass>" > data/secrets/password
``` 
That’s about it, you can start
```bash
docker ocmpose up -d 
```
If everything worked, then   
`curl https://localhost:9000/health`   
will return    
`{"status":"ok"}`   
and you can check provisioners like this   
`curl https://acme-ca.home.arpa:9000/provisioners | jq`   


If not, you need to check what went wrong. 
```bash 
docker compose logs -f
```

## Traefik 

The traefik **docker-compose.yml** file itself requires traefik to trust the root certificate. 
I found several ways on the internet to connect your root certificate in traefik but the simplest and immediately working was adding the variable **LEGO_CA_CERTIFICATES**. I don’t think it makes sense to tell you about the traefik config if you’re already reading this. If not, *[read here](https://doc.traefik.io/traefik/)* .   

```yml
---
services:
  traefik:
    image: traefik
    container_name: traefik
    restart: always
    security_opt:
      - no-new-privileges:true
    ports:
      - 80:80
      - 443:443
    command:
      - "--providers.docker.network=webproxy"
    environment:
      - LEGO_CA_CERTIFICATES=/etc/ssl/certs/root-ca.crt  
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik.yml:/traefik.yml:ro
      - ./data/acme.json:/opt/traefik/acme.json  
      - ./data/custom/:/custom/:ro
      - ./data/ssl/root-ca.crt:/etc/ssl/certs/root-ca.crt:ro  
      - ./data/basic.auth:/basic.auth
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.entrypoints=https"
      - "traefik.http.routers.traefik.rule=Host(`traefik.${DOMAIN}`)"
      - "traefik.http.routers.traefik.tls=true"
      - "traefik.http.routers.traefik.tls.certresolver=stepca"  
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.services.traefik-traefik.loadbalancer.server.port=443"
      - "traefik.http.routers.traefik.middlewares=traefik-auth"
      - "traefik.http.middlewares.traefik-auth.basicAuth.usersFile=/basic.auth"
      - "traefik.http.routers.http-catchall.rule=hostregexp(`{host:.+}`)"
      - "traefik.http.routers.http-catchall.entrypoints=http"
      - "traefik.http.routers.http-catchall.middlewares=redirect-to-https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
    networks:
      - webproxy

networks:
  webproxy:
    name: webproxy
    external: true
```

The only thing, you need to set permissions for the acme.json file   
```bash
chmod 600 acme.json
```


That’s about it, if along the way you wonder what **home.arpa** is, you can check *[rfc8375](https://www.rfc-editor.org/rfc/rfc8375.html)*   . 




