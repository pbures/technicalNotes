# Nginx with TLS termination and LetsEncrypt in Docker

Setting up nginx as the TLS termination for https traffic using letsencrypt as certificate provider.

## Requirements
1. Have docker installed
```
$ docker --version
Docker version 24.0.5, build ced0996
```
1. Have docker-compose installed
```
docker-compose --version
Docker Compose version v2.20.3
```
1. Have a DNS set up, pointing to the server
```
...
Name:	tanks.buresovi.net
Address: 78.80.161.45

```


```
$ sudo useradd -m -U -s /bin/bash webserver
```

## Setup the containers
The letsencrypt provides a tool to receive and autorenew the certificates, .

Docker compose will be used to setup nginx and the letsencrypt tool in separate containers.

1. Create the directory
```
$ mkdir ~/nginx-ssl
```
1. Create the docker-compose file
```
$ vim ~/nginx-ssl/docker-compose.yml
```
with content:
```
version: "3.8"
services:
    web: 
        image: nginx:latest
        restart: always
        volumes:
            - ./public:/var/www/html
            - ./conf.d:/etc/nginx/conf.d
            - ./certbot/conf:/etc/nginx/ssl
            - ./certbot/data:/var/www/certbot
        ports:
            - 80:80
            - 443:443

    certbot:
        image: certbot/certbot:latest
        command: certonly --webroot --webroot-path=/var/www/certbot --email pavel.bures@gmail.com --agree-tos --no-eff-email -d tanks.buresovi.net 
        volumes:
            - ./certbot/conf:/etc/letsencrypt
            - ./certbot/logs:/var/log/letsencrypt
            - ./certbot/data:/var/www/certbot
```
1. create nginx conf file
```
$ mkdir ~/nginx-ssl/conf.d
$ vim ~/nginx-ssl/conf.d/default.conf
```
with content:
```
 server {
     listen [::]:80;
     listen 80;

     server_name domain.com www.domain.com;

     location ~ /.well-known/acme-challenge {
         allow all; 
         root /var/www/certbot;
     }
} 
```

IMPORTANT: Make sure that the nginx server is accessible on port 80. The letsencrypt certbot needs to verify that you are an owner of the domain you with to get the certificate for.
[The ACME protocol](https://tools.ietf.org/html/rfc8555) is used.

In short, the mechanism is that you need to be able to serve a specific challenge from your domain, which is then verified by letsencrypt.

If you encounter any issues, you may want to check the logs, 
```
$ less -S ~/nginx-ssl/certbot/logs/letsencrypt.log
```

Get the certificates, by running the docker images:
```
$ docker-compose up -d
[+] Running 3/3
 ✔ Network nginx-ssl_default      Created                                          0.1s 
 ✔ Container nginx-ssl-certbot-1  Started                                          0.1s 
 ✔ Container nginx-ssl-web-1      Started                                          0.1s 
```

Verify that the certificates were obtained either from logs, or by looking into
```
sudo ls -l ~/nginx-ssl/certbot/conf/live/tanks.buresovi.net/
cert.pem  chain.pem  fullchain.pem  privkey.pem  README
```

## Enable static content serving

Modify the nginx config so the https is accepted, and http is redirected to https.
IMPORTANT: Double check that port 443 is forwarded to your server where you are running the nxing, for instance by port forwarding on the router.

```
$ vim conf.d/default.conf 
```

```
server {
     listen [::]:80;
     listen 80;

     server_name tanks.buresovi.net;

     location ~ /.well-known/acme-challenge {
         allow all;
         root /var/www/certbot;
     }

     return 301 https://tanks.buresovi.net$request_uri;
}

server {
    listen [::]:443 ssl;
    listen 443 ssl;

    server_name tanks.buresovi.net;

    ssl_certificate /etc/nginx/ssl/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/domain.com/privkey.pem;

    root /var/www/html;

    location / {
        index index.html;
    }
}
```

Create a testfile to verify all works so far:
```
$ sudo cat > ~/nginx-ssl/public/index.html
<html>
    <body>
        <h1>Letsencrypt, docker, nginx test file</h1>
    </body>
</html>
```

And restart the containers
```
$ docker-compose restart
```

## Enable forwarding to local http port
This is as simple as changing the location part of the https server

```
server {
    listen [::]:80;
    listen 80;

    server_name tanks.buresovi.net;

    location ~ /.well-known/acme-challenge {
        allow all;
        root /var/www/certbot;
    }

    return 301 https://tanks.buresovi.net$request_uri;
}

server {
    listen [::]:443 ssl;
    listen 443 ssl;

    server_name tanks.buresovi.net;

    ssl_certificate /etc/nginx/ssl/live/tanks.buresovi.net/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/tanks.buresovi.net/privkey.pem;

    #root /var/www/html;

    location / {
        proxy_pass http://<localIP>:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
IMPORTANT: For some reason the dockerized version of the game running on port 3002 does not accept connection on 127.0.0.1

TODO: Investigate and explain. It's not the first time this happened.


## Links
1. https://www.cloudbooklet.com/developer/how-to-install-nginx-and-lets-encrypt-with-docker-ubuntu-20-04/
