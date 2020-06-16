# docker-compose-strapi-
Example Docker-compose App angular and Strapi with Nginx and SSL 

We use the Nginx: Jwilder container to use an nginx reverse. which we enable ports 80 and 443 for ssl.

We create the Volumes to have a persistent Data

```- /var/run/docker.sock:/tmp/docker.sock:ro```

it will connect to the docker socket to do the redirects by querying the environment variables in each container. in our case it will be the URL or domain

```- certs: / etc / nginx / certs: ro```

we keep ssl certificates
 
```- vhostd: /etc/nginx/vhost.d```

We will save the environment variables virtual_host so that letsencryt´s can write on them

```-html: / usr / share / nginx / html```

For the letsencryt's container generate the html to validate the domain

We need the Label tag to make a connection to the jrcs / letsencrypt-nginx-proxy-companion container

Using the jrcs / letsencrypt-nginx-proxy-companion container, we share the volumes 
environment:
``` - NGINX_PROXY_CONTAINER = nginx-proxy```

it will look something like this
```yaml nginx-proxy:
    image: jwilder/nginx-proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - certs:/etc/nginx/certs:ro
      - vhostd:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
    labels:
      - com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy
    networks:
      - strapi-app

  letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    restart: always
    environment:
      - NGINX_PROXY_CONTAINER=nginx-proxy
    volumes:
      - certs:/etc/nginx/certs:rw
      - vhostd:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - strapi-app
```

so create the strapi container, we use postgres
here we add the following environment variables

```VIRTUAL_HOST: $ {VIRTUAL_HOST}``` 
will be our domain

```VIRTUAL_PORT: $ {VIRTUAL_PORT}``` 
we enable the port for the nginx proxy in our case it will be the default value of strapi 1337

```LETSENCRYPT_HOST: $ {LETSENCRYPT_HOST}``` 
will have the same value as the virtual host, and will create the ssl certificate

```LETSENCRYPT_EMAIL: $ {LETSENCRYPT_EMAIL}``` 
mail required for the creation of the ssl certificate

We will use expose to enable the ports internally in docker and that nginx-proxy is in charge of doing the redirects

```expose:'1337'```

PS: we will create the .env file later to save the sensitive variables

 ```yaml strapi:
    image: strapi/strapi
    env_file: .env
    environment:
      #NODE_ENV: production
      DATABASE_CLIENT: ${DATABASE_HOST}
      DATABASE_NAME: ${DATABASE_NAME}
      DATABASE_HOST: ${DATABASE_HOST}
      DATABASE_PORT: ${DATABASE_PASSWORD}
      DATABASE_USERNAME: ${DATABASE_NAME}
      DATABASE_PASSWORD: ${DATABASE_PASSWORD}
      VIRTUAL_HOST: ${VIRTUAL_HOST}
      VIRTUAL_PORT: ${VIRTUAL_PORT}
      LETSENCRYPT_HOST: ${LETSENCRYPT_HOST}
      LETSENCRYPT_EMAIL: ${LETSENCRYPT_EMAIL}
    volumes:
      - ./strapi-app:/srv/app
      - ./strapi-app:/usr/share/nginx/html:ro
    expose:
      - '1337'
    networks:
      - strapi-app
```
   Now it is the turn of our database we add a volume to maintain our databases, it will work on port 5432 and we pass the credentials

```yaml 
    postgres:
     image: postgres
     env_file: .env
     environment:
       POSTGRES_USER: $ {DATABASE_NAME}
       POSTGRES_PASSWORD: $ {DATABASE_PASSWORD}
     volumes:
       - dataDB: / var / lib / postgresql / data
     ports:
       - '5432: 5432'
     networks:
       - strapi-app   
 ```

For this example I will use an angular app that will be our frontend

it works on port 80 and we use expose so that nginx-proxy takes care of redirects
again we add our variables

``` yaml 
VIRTUAL_HOST: domain.com
VIRTUAL_PORT: '80'
LETSENCRYPT_HOST: domain.com
LETSENCRYPT_EMAIL: youremail@email.com 
```

and we will change it for our data

```angular:
    image: raimonxx/mastic:1.0.1
    expose:
      - "80"
    environment:
      VIRTUAL_HOST: domain.com
      VIRTUAL_PORT: '80'
      LETSENCRYPT_HOST: domain.com
      LETSENCRYPT_EMAIL: youremail@email.com
    networks:
      - strapi-app
```

we create the volumes and networks

```yaml
volumes:
  dataDB:
  certs:
  vhostd:
  html:

networks:
  strapi-app:
```
now our docker-compose will look like this

```yaml
  
version: '3.3'

services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - certs:/etc/nginx/certs:ro
      - vhostd:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
    labels:
      - com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy
    networks:
      - strapi-app

  letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    restart: always
    environment:
      - NGINX_PROXY_CONTAINER=nginx-proxy
    volumes:
      - certs:/etc/nginx/certs:rw
      - vhostd:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - strapi-app

  strapi:
    image: strapi/strapi
    env_file: .env
    environment:
      #NODE_ENV: production
      DATABASE_CLIENT: ${DATABASE_HOST}
      DATABASE_NAME: ${DATABASE_NAME}
      DATABASE_HOST: ${DATABASE_HOST}
      DATABASE_PORT: ${DATABASE_PASSWORD}
      DATABASE_USERNAME: ${DATABASE_NAME}
      DATABASE_PASSWORD: ${DATABASE_PASSWORD}
      VIRTUAL_HOST: ${VIRTUAL_HOST}
      VIRTUAL_PORT: ${VIRTUAL_PORT}
      LETSENCRYPT_HOST: ${LETSENCRYPT_HOST}
      LETSENCRYPT_EMAIL: ${LETSENCRYPT_EMAIL}
    volumes:
      - ./strapi-mastic:/srv/app
      - ./strapi-mastic:/usr/share/nginx/html:ro
    expose:
      - '1337'
    networks:
      - strapi-app

  postgres:
    image: postgres
    env_file: .env
    environment:
      POSTGRES_USER: ${DATABASE_NAME}
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
    volumes:
      - dataDB:/var/lib/postgresql/data
    ports:
      - '5432:5432'
    networks:
      - strapi-app

  angular:
    image: raimonxx/mastic:1.0.1
    expose:
      - "80"
    environment:
      VIRTUAL_HOST: domain.com
      VIRTUAL_PORT: '80'
      LETSENCRYPT_HOST: domain.com
      LETSENCRYPT_EMAIL: youremail@email.com
    networks:
      - strapi-app
volumes:
  dataDB:
  certs:
  vhostd:
  html:

networks:
  strapi-app:

```
   
to finish we create in the root of our project ".env" to add our variables or credentials

```
DATABASE_CLIENT=postgres
DATABASE_NAME=strapi
DATABASE_HOST=postgres
DATABASE_PORT=5432
DATABASE_PASSWORD=strapi
VIRTUAL_HOST=sub.domain.com
VIRTUAL_PORT='1337'
LETSENCRYPT_HOST=sub.domain.com
LETSENCRYPT_EMAIL=youremail@email
```

to lift our docker-compose

```cmd 
docker-compose pull
```
unload all containers or use cache

and to lift our docker-compose we use

```cmd 
docker-compose up -d
```

to check the status of our containers
```cmd 
docker ps -a
```


