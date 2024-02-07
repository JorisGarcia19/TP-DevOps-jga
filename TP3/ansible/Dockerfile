# Utilise python comme image de base pour l'image Docker
FROM python:3.8-slim

# Installation de ansible 
RUN pip install ansible

# définit le répertoire de travail dans l'image Docker
WORKDIR /ansible

# Copier les fichiers et dossiers dans l'image Docker
COPY . /ansible

# Pour exécuter le playbook roles.yml 
CMD ["ansible-playbook", "-i", "inventories/setup.yml", "roles.yml"]