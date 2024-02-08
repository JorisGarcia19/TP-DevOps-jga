# TP 3

## Introduction

### Inventories

Je crée le fichier `setup.yml` dans `TP3/ansible/inventories/setup.yml`

```yml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: /home/joris/.ssh/id_rsa_ansible
 children:
   prod:
     hosts: joris.garcia.takima.cloud
```

Je vais dans le répertoire : `ansible` et j'exécute cette commande :

```yml
ansible all -i inventories/setup.yml -m ping
```

Le résultat du ping :

```yml
joris.garcia.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

### Facts

Pour demander au serveur la distribution :

```shell
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

Le résulat :

```shell
joris.garcia.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "CentOS",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/redhat-release",
        "ansible_distribution_file_variety": "RedHat",
        "ansible_distribution_major_version": "7",
        "ansible_distribution_release": "Core",
        "ansible_distribution_version": "7.9",
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}
```

Pour supprimer le serveur httpd :

```shell
ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become
```

Le résultat :

```shell
joris.garcia.takima.cloud | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "removed": [
            "httpd"
        ]
    },
    "msg": "",
    "rc": 0,
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-99.el7.centos.1 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-99.el7.centos.1         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-99.el7.centos.1.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-99.el7.centos.1.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-99.el7.centos.1                                          \n\nComplete!\n"
    ]
}
```

3-1 Document your inventory and base commands :

```shell
all: #  désigne tous les hôtes et groupes définis dans cet inventaire.
 vars: # Définir des variables applicables à tous les hôtes 
   ansible_user: centos # spécifie le nom d'utilisateur à utiliser lors de la connexion aux hôtes. 'centos' est l'utilisateur pour toutes les connexions SSH.
   ansible_ssh_private_key_file: /home/joris/.ssh/id_rsa_ansible # indique le chemin d'accès au fichier de clé privée SSH utilisé pour se connecter aux hôtes.
 children: # permet de définir des groupes d'hôtes.
   prod: # est un groupe d'hôtes
     hosts: joris.garcia.takima.cloud # liste les adresses des hôtes appartenant à ce groupe.
```

## Playbooks

### First playbook

Je crée un playbook dans `TP3/ansible/playbook.yml`.

```yml
- hosts: all
  gather_facts: false
  become: true

  tasks:
   - name: Test connection
     ping:
```

Je vérifie la syntaxe avec :

```shell
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check
```

Pour tester le playbook.yml :

```shell
ansible-playbook -i inventories/setup.yml playbook.yml
```

Le resultat :

```shell
PLAY [all] *************************************************************************************************************************************************

TASK [Test connection] *************************************************************************************************************************************
ok: [joris.garcia.takima.cloud]

PLAY RECAP *************************************************************************************************************************************************
joris.garcia.takima.cloud  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

### Advanced Playbook

Je crée donc un nouveau playbook : `docker.yml` qui va installer docker.

```yml
- hosts: all
  gather_facts: false
  become: true

# Install Docker
  tasks:
  - name: Install device-mapper-persistent-data
    yum:
      name: device-mapper-persistent-data
      state: latest

  - name: Install lvm2
    yum:
      name: lvm2
      state: latest

  - name: add repo docker
    command:
      cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker
    yum:
      name: docker-ce
      state: present

  - name: Install python3
    yum:
      name: python3
      state: present

  - name: Install docker with Python 3
    pip:
      name: docker
      executable: pip3
    vars:
      ansible_python_interpreter: /usr/bin/python3

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker
```

je vérifie la syntaxe :

```shell
ansible-playbook --syntax-check -i inventories/setup.yml docker.yml
```

Pour exécuter le playbook :

```shell
ansible-playbook  -i inventories/setup.yml docker.yml
```

Le resultat :

```shell
PLAY RECAP *************************************************************************************************************************************************
joris.garcia.takima.cloud  : ok=7    changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Using roles

Pour initialiser un nouveau rôle (`docker`) dans le système de gestion de configuration Ansible :

```shell
ansible-galaxy init roles/docker
```

Les répertoires sont créés :

![alt text](./images/image-1.png)

Je garde seulement le répertoire `tasks` et `handlers`.

![alt text](./images/image-2.png)

Je crée un autre playbook `roles.yml` pour tester le role :

```yml
# Nom du playbook
- name: roles
  # Spécifie que ce playbook s'applique à tous les hôtes
  hosts: all
  # Active les privilèges d'administration
  become: true
  # liste des rôles Ansible à appliquer
  roles:
    - roles/docker
```

Pour vérifier la syntaxe du playbook.

```shell
ansible-playbook --syntax-check -i inventories/setup.yml roles.yml 
```

Pour lancer le playbook :

```shell
 ansible-playbook  -i inventories/setup.yml roles.yml
 ```

Je supprime le playbook `docker.yml` pour mettre son contenu dans le fichier `main.yml` du repertoire `tasks` du role docker.

En déclarant le role docker dans le playbook `role.yml` et mettant les commandes pour installer docker dans `tasks`, le playbook `docker.yml` devient alors inutiles.

## Deploy your App

```yml
# Fichier tasks pour install_docker
# Installe le paquet nécessaire pour la gestion des volumes logiques
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest
# Installe lvm2, nécessaire pour la gestion des volumes logiques
- name: Install lvm2
  yum:
    name: lvm2
    state: latest

# Ajoute le dépôt officiel de Docker pour CentOS
- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

# Installe Docker Community Edition
- name: Install Docker
  yum:
    name: docker-ce
    state: present

# Installe Python 3
- name: Install python3
  yum:
    name: python3
    state: present

 # Installe le module Docker pour Python pour permettre la gestion de Docker via des scripts Python
- name: Install docker with Python 3
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

# Assure que le service Docker est démarré et actif
- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```

Je crée les autres roles :

```shell
ansible-galaxy init roles/create_network
ansible-galaxy init roles/launch_database
ansible-galaxy init roles/launch_app
ansible-galaxy init roles/launch_proxy
```

J'ajoute mes roles dans le playbook des roles `roles.yml` :

```yml
- name: roles
  hosts: all
  become: true
  vars_files:
    - group_vars/all.yml
  roles:
    - role: install_docker
    - role: create_network
    - role: launch_database
    - role: launch_app
    - role: launch_proxy

```

Je configure les tâches de mes roles :

```yml
# Fichier tasks pour create_network
- name: Create Docker app-network 
  docker_network:
    # Créer le réseau s'il n'existe pas 
    name: "{{ DOCKER_NETWORK }}"
    state: present
```

```yml
# Fichier tasks pour launch_app
- name: Run Api
  docker_container:
    # Définition du nom du conteneur Docker 
    name: api
    # Spécification de l'image Docker à utiliser
    image: "{{ DOCKER_NAME }}/tp-devops-api:1.0"
    # Connecte le conteneur à un réseau Docker
    networks:
      - name: "{{ DOCKER_NETWORK }}"
    # Spécification des variables d'environnement
    env:
      POSTGRES_DB: "{{ DATABASE_DB }}"
      POSTGRES_USER: "{{ DATABASE_USER }}"
      POSTGRES_PASSWORD: "{{ DATABASE_PASSWORD }}"
      DOCKER_NAME_POSTGRESQL: "{{ DOCKER_NAME_POSTGRESQL }}"
      POSTGRESQL_PORT: "{{ POSTGRESQL_PORT }}"
```

```yml
# Fichier tasks pour launch_database
- name: Run postgresql
  docker_container:
    # Définition du nom du conteneur Docker 
    name: "{{ DOCKER_NAME_POSTGRESQL }}"
    # Spécification de l'image Docker à utiliser
    image: "{{ DOCKER_NAME }}/tp-devops-postgresql:latest"
    # Connecte le conteneur à un réseau Docker
    networks:
      - name: "{{ DOCKER_NETWORK }}"
    # Spécification des variables d'environnement
    env:
      POSTGRES_DB: "{{ DATABASE_DB }}"
      POSTGRES_USER: "{{ DATABASE_USER }}"
      POSTGRES_PASSWORD: "{{ DATABASE_PASSWORD }}"
```

```yml
# Fichier tasks du proxy 
- name: Run HTTPD
  docker_container:
    # Définition du nom du conteneur Docker 
    name: httpd
    # Spécification de l'image Docker à utiliser
    image: "{{ DOCKER_NAME }}/tp-devops-serveur:latest"
    # Mappe le port 80 du conteneur sur le port 80 de l'hôte, permettant l'accès au service HTTPD depuis l'extérieur
    ports:
      - "80:80"
    # Connecte le conteneur à un réseau Docker
    networks:
      - name: "{{ DOCKER_NETWORK }}"
```

J'ai crée un fichier qui va contenir les variables d'environnement (`all.yml`).

```yml
DATABASE_DB: "db"
DATABASE_USER: "usr"
DATABASE_PASSWORD: "mdp"

DOCKER_NAME: "jorisgarcia"
DOCKER_NETWORK: "app-network"
```

## Continuous Deployment

Pour déployer en continu l'application, il faut mettre en place un nouveau workflow : `deploy.yml`.

Pour sécuriser les variables, il faut utiliser un `ansible-vault`. Il va permettre de chiffrer les variables de `all.yml`.
Il faut utiliser cette commande et saisir un mot de passe :

```yml
ansible-vault encrypt group_vars/all.yml
```

Le resultat :

```yml
$ANSIBLE_VAULT;1.1;AES256
37353765613339646435656634346563613836376562356332393562623939356130336266646234
3166303935313764653065653530393832666432353566610a366466616534316437396564373866
63386334663734313661383862316432636636323636613466623934353665326538666339613531
6636356263306163300a366237323230316666303435643531613531323639626132626239303634
61383836366232653230353030346362396165653736323238633961393861333239356534646562
30393835346539303338613638373537653764626236353432333334316235323861323138303833
33316639356461343432383632396431333530363736666465633133336264633435303131656636
61333464373232346135366465663538636136643761353534306137386131343065376532363239
33343935396630356663373363376239353665303837616564643365653935336366643035343132
61373262626136343632393961623566613763363463326465323637303265326331626632356166
64376132333134356536663039313966306164393261346362373766666266363061323035333735
66613736373065653561626564333961346661633063333139623862346262623533383063373830
65353963636132613362343466343237626265386138613634633261616364323866
```

J'ajoutee donc un répertoire secret sur github `ANSIBLE_VAULT_PASSWORD` qui contient le mot de passe du `ansible-vault`.

Je stocke également dans un répertoire secret la clé ssh et l'utilisateur d'ansible : `SSH_PRIVATE_KEY`, `ANSIBLE_USER`.

Je crée un workflow `deploy.yml` qui va permettre de déployer en continu l'application :

```yml
name: deploy

on:
  # Lancer le workflow si l'action a eu lieu sur la branche main et que le workflow buildpush est terminé
  workflow_run:
    workflows: [buildpush]
    types: 
      - completed
    branches: 
      - main

jobs:
  deploy:
    # Plateforme d'exécution
    runs-on: ubuntu-22.04

    # Lancer les etapes si le worflow précédent n'a pas échoué 
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:

    # Pour récupérer le code du repository.
    - name: Checkout code
      uses: actions/checkout@v2.5.0

    # Exécuter un playbook Ansible
    - name: Run Ansible playbook
      uses: dawidd6/action-ansible-playbook@v2
      with:
        # Spécification du chemin vers le playbook à exécuter
        playbook:  ./TP3/ansible/roles.yml
        # Spécification de la clé SSH privée stockée dans les secrets GitHub pour l'authentification
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        # Spécification du mot de passe du coffre pour déchiffrer les variables sécurisées
        vault_password: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}
        # Définition de l'inventaire avec un groupe 'all' et l'adresse d'hôte
        inventory: |
          [all]
          joris.garcia.takima.cloud
        # Spécification de l'utilisateur Ansible pour la connexion SSH
        options: |
          -u${{secrets.ANSIBLE_USER}}

    - name: Notify deployment success
      run: echo "Deployment successful!"
```
