# TP 1

## Database

Création du Dockerfile pour postgresql

```yml
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd
```

Build l'image :

```yml
docker build -t app/postgresql .
```

### Init database

Création d'un réseau :

```yml
docker network create app-network
```

Relance le conteneur :

```yml
docker run --name postgresql -p 5432:5432 --network app-network -e POSTGRES_PASSWORD="mdp" -d app/postgresql
```

Lance adminer :

```yml
docker run -d --network app-network -p 8080:8080 adminer
```

Je rajoute cette ligne dans le Dockerfile de postgresql afin de copier les scripts d'init dans le repertoire `docker-entrypoint-initdb.d`

```yml
COPY ./sql-scripts/* /docker-entrypoint-initdb.d/
```

Je build a nouveau l'image :

```yml
docker build -t app/postgresql .
```

```yml
docker run --name postgresql -p 5432:5432 --network app-network -d -e POSTGRES_PASSWORD="mdp" app/postgresql
```

### Persist data

Ajout d'un volume pour sauvegarder les données

```yml
docker run --name postgresql -p 5432:5432 --network app-network  -v ./data:/var/lib/postgresql/data -e POSTGRES_PASSWORD="mdp" -d app/postgresql
```

Desormais les informations modifiées sont sauvegardées

1-1 :

- commandes :

  - Build :

    ```yml
    docker build -t app/postgresql .
    ```

  - Run :

    ```yml
    docker run --name postgresql -p 5432:5432 --network app-network  -v ./data:/var/lib/postgresql/data -e POSTGRES_PASSWORD="mdp" -d app/postgresql
    ```

- Dockerfile :

    ```yml
        FROM postgres:14.1-alpine
    
        ENV POSTGRES_DB=db \
        POSTGRES_USER=usr \
        POSTGRES_PASSWORD=pwd
    
        COPY ./scripts/* /docker-entrypoint-initdb.d/
    ```

## API

### Basic

```yml
FROM eclipse-temurin:21

WORKDIR /app

COPY Main.class /app

CMD ["java", "Main"]
```

```yml
docker build -t app/api .
```

```yml
docker run --name api -d app/api
```

### Multistage build

1-2 Why do we need a multistage build? And explain each step of this dockerfile.

Backend api :

je modifie le fichier `application.yml` :

```yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://postgresql:5432/db
    username: usr
    password: ${MDP}
    driver-class-name: org.postgresql.Driver
management:
 server:
   add-application-context-header: false
 endpoints:
   web:
     exposure:
       include: health,info,env,metrics,beans,configprops
```

```yml
docker build -t  app/api .
```

```yml
docker run --name api -p 8081:8080 -e MDP="mdp" --network app-network -d app/api
```

Recuperer le fichier de config :

```yml
docker cp serveur:/usr/local/apache2/conf/httpd.conf ./httpd.conf
```

## Http server

### Basics

J'utilise un serveur apache :

```yml
FROM httpd:2.4
```

### Configuration

Pour récupérer le fichier de conf :

```yml
docker cp serveur /usr/local/apache2/conf/httpd.conf
```

Pour que les modifications du fichier httpd.conf soient prises en compte je rajoute cette ligne dans le dockerfile :

```yml
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```

## Reverse proxy

Pour configurer le reverse proxy, je modifie le fichier httpd.conf :

```yml
ServerName localhost
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://api:8080/
ProxyPassReverse / http://api:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

Un reverse proxy permet d'améliorer la sécurité, l'équilibrage de charge, la performance et la fiabilité d'un système informatique. Il agit comme un intermédiaire entre les utilisateurs et les serveurs, permettant ainsi de gérer le trafic entrant.

## Docker-compose

Je configure les différents services dans le `docker-compose.yml`.*

```yml
version: "3.7"

services:
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    environment:
      - NAME_DB=${DB_NAME}
      - USER_DB=${DB_USER}
      - MDP_DB=${DB_PWD}
    networks:
      - app-network
    depends_on:
      - postgresql

  postgresql:
    build:
      context: ./postgresql
      dockerfile: Dockerfile
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PWD}
    volumes:
      - ./postgresql/data:/var/lib/postgresql/data
    networks:
      - app-network

  httpd:
    build:
      context: ./serveur
      dockerfile: Dockerfile
    ports:
      - "80:80"
    networks:
      - app-network
    depends_on:
      - postgresql
      - api

networks:
  app-network:
```

1.3 Commandes les plus importantes :

- docker-compose up :
        Démarre les services définis dans le fichier docker-compose.yml. Si le service n'a pas été construit auparavant, docker-compose up le construira automatiquement.
- docker-compose down :
        Arrête et supprime les ressources créées par docker-compose up.

1.4 Documenter le fichier docker-compose :

- services: correspond aux différents conteneurs

- ``build``: est la section qui va permettre la construction des images car elle contient les informations sur le repertoire qui contient le dockerfile (context) et le nom du dockerfile à utiliser (dockerfile)

- ``environment``: permet de définir des variables d'environnement dans les conteneurs

- ``volumes``: permet de créer un volume pour sauvegarder des données  

- ``networks``: permet de spécifier les réseaux du conteneur

- ``depends_on``: permet de définir des dépendances entre les services.

- ``networks``: permet de définir et configurer des réseaux personnalisés

## Publish

Pour publier une image, il faut lui ajouter un tag :

```yml
docker tag app-postgresql jorisgarcia/jga-postgresql:1.0
```

Une fois taguer, on peut publier l'image sur le dockerhub

```yml
docker push jorisgarcia/jga-postgresql:1.0
```
