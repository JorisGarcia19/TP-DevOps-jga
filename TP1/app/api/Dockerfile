# # Utiliser une image eclipse
# FROM eclipse-temurin:21
# # Créer un répertoire pour l'application dans le conteneur
# WORKDIR /app
# # Copier le fichier Main.class dans le conteneur
# COPY Main.class /app
# # Commande pour exécuter votrlae classe Java
# CMD ["java", "Main"]

# Build
FROM maven:3.9.6-amazoncorretto-21 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:21
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar 
