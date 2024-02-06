# TP 2

2-1 Qu'est ce qu'un conteneur de test ?

Les conteneurs de test sont des conteneurs Docker pour les tests d'intégration. Ils permettent d'exécuter des tests dans des environnements isolés et contrôlables.

## Build and test your Application

`mvn clean verify`
Que fait cette commande ?

Cette ligne permet de vérifier et de nettoyer le projet en supprimant les anciens artefacts et vérifie ensuite que le code actuel est non seulement compilable, mais aussi qu'il passe tous les tests, y compris les tests d'intégration.

2-2 Document your Github Actions configurations :

Je crée le répertoire worflow et le fichier main.yaml :

```yml
name: CI devops 2023
on:
  push:
    # Spécifier les branches où le workflow se déclenchera pour les événements de push
    branches: 
      - main
      - develop

jobs:
  test-backend: 
    # Plateforme d'exécution
    runs-on: ubuntu-22.04
    steps:
      # Permet de télécharger le code du dépôt GitHub dans l'environnement d'exécution du workflow
      - uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
        # configurer l'environnement d'exécution Java 
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build and test with Maven
        # definit le repertoire de travail
        working-directory: ./TP1/app/api
        # permet de vérifier et nettoyer l'environnement de build en supprimant les artefacts de builds précédents
        run: mvn clean verify
```

## First steps into the CD World

1.Add your docker hub credentials to the environment variables in GitHub Actions (and let them secured).

Je vais dans les paramètres de github puis dans secrets, je crée deux répertoires secrets afin de stocker de manière sécurisée les informations de connexions pour docker.

* DOCKERHUB_USERNAME
* DOCKERHUB_TOKEN

 Je vais sur docker hub et me connecte. Je vais dans les paramètres pour générer un token et le renseigner dans le répertoire `DOCKERHUB_TOKEN`

 2.Build your docker images inside your GitHub Actions pipeline.

Je modifie le fichier main.yml pour ajouter le job `build-and-push-docker-image`. Il va donc construire les images et ajouter un tag aux images.

```yml
build-and-push-docker-image:
   needs: test-backend
   # Plateforme d'exécution
   runs-on: ubuntu-22.04
  
   # Les différentes etapes pour construire et publier les images  
   steps:
     - name: Checkout code
       uses: actions/checkout@v2.5.0

     # Etape pour construire l'image de la base de l'api 
     - name: Build image and push backend
       uses: docker/build-push-action@v3
       with:
         # Chemin relatif où se trouve le code source avec le Dockerfile
         context: ./TP1/app/api
         # Crée un tag pour l'image Docker
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-api:latest

     # Etape pour construire l'image de la base de données   
     - name: Build image and push database
       uses: docker/build-push-action@v3
       with:
         # Chemin relatif où se trouve le code source avec le Dockerfile
         context: ./TP1/app/postgresql
         # Crée un tag pour l'image Docker
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-postgresql:latest
    
     # Etape pour construire et publier l'image du reverse proxy 
     - name: Build image and push httpd
       uses: docker/build-push-action@v3
       with:
         # Chemin relatif où se trouve le code source avec le Dockerfile
         context: ./TP1/app/serveur
         # Crée un tag pour l'image Docker
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-serveur:latest
```

Je commit et je push. Les tests sont validés.

3.Publish your docker images when there is a commit on the main branch.

Je modifie le `main.yml` afin d'ajouter une étape qui se connecte à dockerhub.

```yml
# Etape pour la connexion à DockerHub
- name: Login to DockerHub
  run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}
```

Cela permet de se connecter à docker et ainsi de pouvoir publier les images.

Je rajoute cette ligne pour chaque étape afin de publier les images générées.

```yml
push: ${{ github.ref == 'refs/heads/main' }}
```

## Setup Quality Gate

Pour mettre en place la quality gate il faut se connecter sur sonar et créer une organisation.
Il faut également créer un répertoire secret sur github pour stocker le token de sonar.

Je modifie donc la ligne qui exécute la commande de maven afin d'intégrer l'analyse SonarCloud.

```yml
run: mvn -B verify sonar:sonar -Dsonar.projectKey=tp-devops-jga_tp-devops-jga -Dsonar.organization=tp-devops-jga -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml
```

## Bonus: split pipelines (Optional)

Pour que le test `test-backend` soit lancé lorsqu'il y a des actions sur les branches `develop` et `master`, mais également pour que le test `build-and-push-docker-image` ne soit lancé que lorsqu'il y a des actions sur `master`, j'ai décidé de découper le fichier `main.yml` en deux fichiers. Le premier main.yml ne contient que le test `test-backend`. Le second `buildpush.yml` contient seulement `build-and-push-docker-image`. Cette configuration permet de spécifier uniquement `master` comme branche dans `buildpush.yml`, ainsi le workflow ne va se lancer uniquement lorsqu'il y aura des actions sur `master`.

```yml
# main.yml
name: test-backend
on:
  push:
   # Spécifier les branches où le workflow se déclenchera pour les événements de push
    branches: 
      - main
      - develop
```

```yml
# buildpush.yml
name: buildpush
on:
  workflow_run:
    workflows: [test-backend]
    types: 
      - completed
    branches: [main] 
```

Cette configuration permet également d'utiliser `workflows_run` ce qui permet d'exécuter le test `build-and-push-docker-image` lorsque le test `test-backend` est complété.

Pour éviter de lancer le `build-and-push-docker-image` si le test précèdent a échoué, j'ai également rajouté cette ligne :

```yml
if: ${{ github.event.workflow_run.conclusion == 'success' }}
```
