# About

This example will demonstrate using nginx to proxy requests to a http service.
nginx is terminating ssl in this request.

*upsteam host*  host running service nginx will proxy connections too
*proxy host*  host running nginx
*proxy dns record* dns A record pointing to proxy host/network address (ie proxy-A.record.com)

## Files

```
├── Dockerfile # docker image used to run nginx
├── docker-compose.yml # docker container runtime settings
├── nginx.conf # nginx stanza to proxy requests to an upstream http service
└── nginx.env # enviroment variable used to template nginx.conf
```

## Prerequistes

- host with docker/docker-compose installed
- ingress between upstream and proxy hosts established
- valid dns record with assigned ip address your *proxy host*
- valid ssl certs present on the host

## Setup

### Seed  enviroment variables (nginx.env)
```
tee -a nginx.env << END
HTTP_SERVICE_PORT=80
HTTP_SERVICE=<upsteam host private ip>
END
```

### Add ssl certificates (nginx.conf)

example using certbot
```
    ssl_certificate /etc/letsencrypt/live/ccureproxy.stagevoy.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/ccureproxy.stagevoy.com/privkey.pem; # managed by Certbot
```

additionally set your server_name == proxy dns record
```
    server_name << proxy-A.record.com >>;
```

### Add ssl volume mount (docker-compose.yml)

include your host ssl certs
```
    volumes:
      - /etc/letsencrypt/:/etc/letsencrypt/:rw
```

### run nginx

```
docker-compose up
```

### query your proxy

```
wget https://proxy-A.record.com
nc -zv proxy-A.record.com 443
```

### versions used
- Docker version 20.10.12, build e91ed57
- docker-compose version 1.29.2
- macOS 12.0.1 (21A559)
- zsh 5.8 (x86_64-apple-darwin21.0)

### certbot install
```
 apk add certbot python3 python3-dev py3-pip 
 certbot --nginx -d << proxy-A.record.com  -d proxy-A.record.com 
```

# Resources
- https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/
- https://geko.cloud/en/nginx-letsencrypt-certbot-docker-alpine/
- https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/