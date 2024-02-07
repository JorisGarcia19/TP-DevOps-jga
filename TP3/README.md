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

Je vais dans le repertoire : `ansible` et j'execute cette commande :

```yml
ansible all -i inventories/setup.yml -m ping
```

Le resultat du ping :

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

Le resulat :

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
   ansible_user: centos # 'spécifie le nom d'utilisateur à utiliser lors de la connexion aux hôtes. 'centos' est l'utilisateur pour toutes les connexions SSH.
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

Je verifie la syntaxe avec :

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

```
