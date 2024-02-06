# # Utiliser une image eclipse
# FROM eclipse-temurin:21
# # Créer un répertoire pour l'application dans le conteneur
# WORKDIR /app
# # Copier le fichier Main.class dans le conteneur
# COPY Main.class /app
# # Commande pour exécuter votrlae classe Java
# CMD ["java", "Main"]

# Build
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
