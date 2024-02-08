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

Pour lancer le conteneur :
En précisant les ports de la machine et du conteneur avec `-p` et en donnant une variable d'environnement pour le mot de passe avec `-e`.

```yml
docker run --name postgresql -p 5432:5432  -e POSTGRES_PASSWORD="mdp" -d app/postgresql
```

### Init database

Création d'un réseau :

```yml
docker network create app-network
```

Cela va permettre à plusieurs conteneurs de communiquer.

Je relance le conteneur :
En précisant cette fois-ci le reseau avec `--network`  

```yml
docker run --name postgresql -p 5432:5432 --network app-network -e POSTGRES_PASSWORD="mdp" -d app/postgresql
```

Commande pour lancer adminer :

```yml
docker run -d --network app-network -p 8080:8080 adminer
```

Je rajoute cette ligne dans le Dockerfile de postgresql afin de copier les scripts d'init dans le repertoire `docker-entrypoint-initdb.d`. La base de données sera ainsi initialisé avec les scripts.

```yml
COPY ./sql-scripts/* /docker-entrypoint-initdb.d/
```

Il faut build à nouveau l'image et relancer le conteneur :

```yml
docker build -t app/postgresql .
```

```yml
docker run --name postgresql -p 5432:5432 --network app-network -d -e POSTGRES_PASSWORD="mdp" app/postgresql
```

### Persist data

Ajout d'un volume pour sauvegarder les données.

```yml
docker run --name postgresql -p 5432:5432 --network app-network  -v ./data:/var/lib/postgresql/data -e POSTGRES_PASSWORD="mdp" -d app/postgresql
```

Désormais les informations modifiées sont sauvegardées.

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
        # Utilise l'image de PostgreSQL version 14.1 basée sur Alpine Linux comme base
        FROM postgres:14.1-alpine

        # Définition des variables d'environnement pour la configuration de PostgreSQL
        ENV POSTGRES_DB=db \
        POSTGRES_USER=usr 

        # Copier les scripts d'initiation depuis le répertoire local vers /docker-entrypoint-initdb.d/ dans le conteneur. 
        COPY ./scripts/ /docker-entrypoint-initdb.d/
    ```

## API

### Basic

```yml
# Utiliser  l'image eclipse-temurin version 21
FROM eclipse-temurin:21

# Définir le répertoire de travail dans le conteneur
WORKDIR /app

# Copier le fichier Main.class dans le conteneur
COPY Main.class /app

# Commande pour exécuter la classe Java
CMD ["java", "Main"]
```

Pour executer le code Java, il faut build l'image et lancer le conteneur.

```yml
docker build -t app/api .
```

```yml
docker run --name api -d app/api
```

### Multistage build

1.Backend simple api

```yml
# Utilise comme image de base maven version 3.9.6
FROM maven:3.9.6-amazoncorretto-21 AS myapp-build # AS myapp-build : donne un nom à cette étape de build

# Définit une variable d'environnement qui indique le répertoire de l'application dans l'image.
ENV MYAPP_HOME /opt/myapp

# Définit le répertoire de travail courant dans l'image
WORKDIR $MYAPP_HOME

# Copie le fichier pom.xml du dossier source local dans le répertoire de travail de l'image Docker.
COPY pom.xml .

# Copie les sources de l'application dans le répertoire de travail de l'image Docker.
COPY src ./src

# Exécute la commande Maven pour construire le projet tout en sautant l'exécution des tests.
RUN mvn package -DskipTests

# Run
# Utilise amazoncorretto comme image de base pour l'étape
FROM amazoncorretto:21
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME

# Copie le fichier JAR produit par Maven lors de l'étape de build
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# Définit le point d'entrée de l'image Docker et exécuter l'application Java
ENTRYPOINT java -jar myapp.jar
```

1-2 Pourquoi avons-nous besoin de multistage build?

Le multistage build permet de séparer les étapes de construction et d'exécution, en utilisant des images de base différentes pour chaque étape. Ainsi, les outils et dépendances nécessaires uniquement lors de la construction ne se retrouvent pas dans l'image finale, réduisant sa taille et minimisant les risques de sécurité.

2.Backend api :

Je modifie le fichier `application.yml` :
Je renseigne le `username`, l'url et le mot de passe par le biais d'une variable d'environnement.

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

Il faut build l'image et lancer le conteneur.

```yml
docker build -t  app/api .
```

Il ne faut pas oublier le reseau dans le run pour que l'api puisse communiquer avec postgresql.

```yml
docker run --name api -p 8081:8080 -e MDP="mdp" --network app-network -d app/api
```

## Http server

### Basics

Recuperer le fichier de config :

```yml
docker cp serveur:/usr/local/apache2/conf/httpd.conf ./httpd.conf
```

J'utilise un serveur apache :

```yml
FROM httpd:2.4
# Copier index.html dans l'image Docker 
COPY ./index.html /usr/local/apache2/htdocs/
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

Pour configurer le reverse proxy, je modifie le fichier `httpd.conf` :

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

Ce code permet de configurer un reverse proxy qui écoute sur le port 80 et transfère les requêtes à l'API qui écoute sur le port 8080.

Un reverse proxy permet d'améliorer la sécurité, l'équilibrage de charge, la performance et la fiabilité d'un système informatique. Il agit comme un intermédiaire entre les utilisateurs et les serveurs, permettant ainsi de gérer le trafic entrant.

## Docker-compose

Je configure les différents services dans le `docker-compose.yml`.*

```yml
version: "3.8"

services:
  api:
    build:
      context: ./api
    env_file:
      - ./.env
    environment:
      - POSTGRES_DB
      - POSTGRES_USER
      - POSTGRES_PASSWORD
    networks:
      - app-network
      - app-proxy
    depends_on:
      - postgresql

  postgresql:
    build:
      context: ./postgresql
    env_file:
      - ./.env
    environment:
      - POSTGRES_DB
      - POSTGRES_USER
      - POSTGRES_PASSWORD
    volumes:
      - postgresql:/var/lib/postgresql/data
    networks:
      - app-network

  httpd:
    build:
      context: ./serveur
    ports:
      - "80:80"
    networks:
      - app-proxy
    depends_on:
      - api

networks:
  app-network:
  app-proxy:

volumes:
  postgresql:
```

1.3 Commandes les plus importantes :

- docker-compose up :
        Démarre les services définis dans le fichier `docker-compose.yml`. Si le service n'a pas été construit auparavant, docker-compose up le construira automatiquement.
- docker-compose down :
        Arrête et supprime les ressources créées par docker-compose up.

1.4 Documenter le fichier docker-compose :

- ``services``: correspond aux différents conteneurs

- ``build``: est la section qui va permettre la construction des images car elle contient les informations sur le repertoire qui contient le dockerfile (context)

- ``environment``: permet de définir des variables d'environnement dans les conteneurs

- ``volumes``: permet de créer un volume pour sauvegarder des données et éviter de perdre des données qui ont été modifiées.

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
