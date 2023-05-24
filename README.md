# Dockerized HAProxy with Let's Encrypt automatic certificates

This container provides an HAProxy instance with Let's Encrypt certificates generated
at startup, as well as renewed (if necessary) once a week with an internal cron job.

## Usage

### Build images from Dockerfiles:

The first image that needs to be built is the NGinx ones, that will host the index.html file:
```
cd web-server && docker build -t docker-haproxy-certbot:latest .
```

Then, we need to build the haproxy-certbot image, that will host the HA Proxy + Certbot to auto-renew the certificates:
```
docker build -t docker-haproxy-certbot:latest .
```

### Run container:

Example of run command (replace CERTS,EMAIL values and volume paths with yours)

```
docker run --name lb -d \
    -e CERTS=my.domain \
    -e EMAIL=my.email@my.domain \
    -e STAGING=true \
    -v '$PWD/data/letsencrypt:/etc/letsencrypt' \
    -v '$PWD/data/haproxycfg/haproxy.cfg:/etc/haproxy/haproxy.cfg' \
    --network my_network \
    -p 80:80 -p 443:443 \
    docker-haproxy-certbot:latest
```

### Run with docker-compose:

Use the docker-compose.yml file (it will create creates 5 containers, the haproxy, rsyslog and 3 nginx container linked in haproxy configuration)

```
# docker-compose.yml file content:

version: '3'
services:
    haproxy:
        container_name: lb
        environment:
            - CERTS=mydomain.com
            - EMAIL=my.email@my.domain
            - STAGING=true
        volumes:
            - '$PWD/data/letsencrypt:/etc/letsencrypt'
            - '$PWD/data/haproxy.cfg:/etc/haproxy/haproxy.cfg'
        networks:
            - alyantNet
        ports:
            - '80:80'
            - '443:443'
        image: 'docker-haproxy-certbot:latest'
    rsyslog:
        container_name: rsyslog
        environment:
            - TZ=UTC
        volumes:
            - '$PWD/data/rsyslog/config:/config'
        networks:
            - alyantNet
        ports:
            - '514:514'
        image: 'rsyslog/syslog_appliance_alpine'
    back01:
        container_name: lorem-ipsum-01
        networks:
            - alyantNet
        image: 'alyant-lorem-ipsum:latest'
    back02:
        container_name: lorem-ipsum-02
        networks:
            - alyantNet
        image: 'alyant-lorem-ipsum:latest'
    back03:
        container_name: lorem-ipsum-03
        networks:
            - alyantNet
        image: 'alyant-lorem-ipsum:latest'
networks:
  alyantNet:
  

$ docker-compose up -d

```

### Customizing Haproxy

You will almost certainly want to create an image `FROM` this image or
mount your `haproxy.cfg` at `/etc/haproxy/haproxy.cfg`.

    docker run [...] -v <override-conf-file>:/etc/haproxy/haproxy.cfg docker-haproxy-certbot:latest

The haproxy configuration provided file comes with the "resolver docker" directive to permit DNS runt-time resolution on backend hosts (see https://github.com/gesellix/docker-haproxy-network).

### Scaling UP and DOWN

If you need to scale it to more NGinx servers, for example, you will need to add a new set of properties for that new container, inside of docker-compose.yml, for example, if we want to a 4th NGinx container, just add it before the "networks" directive:

```
    back04:
        container_name: lorem-ipsum-04
        networks:
            - alyantNet
        image: 'alyant-lorem-ipsum:latest'
```

And add a new line inside of the haproxy.cfg file:
```
...
server backend4 lorem-ipsum-04:80 check resolvers docker resolve-prefer ipv4
...
```

### Renewal cron job

Once a week a cron job check for expiring certificates with certbot agent and reload haproxy if a certificate is renewed. No containers restart needed.
