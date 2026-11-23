# Ansible Infrastructure Automation

Projet d'automatisation destiné au déploiement et à l'administration de serveurs Ubuntu 24.04 LTS sous VMware.

L'objectif est de provisionner rapidement de nouvelles machines virtuelles à partir d'un template standardisé puis de déployer automatiquement les configurations et services nécessaires à l'aide d'Ansible.

---

# Objectifs

- Standardiser les déploiements Linux.
- Réduire les opérations manuelles répétitives.
- Garantir une configuration cohérente entre les serveurs.
- Faciliter les migrations vers Ubuntu 24.04 LTS.
- Documenter l'infrastructure sous forme de code (Infrastructure as Code).

---

# Architecture

```text
Template Ubuntu 24.04
        │
        ▼
Clonage VMware
        │
        ▼
Configuration IP
        │
        ▼
Ajout dans l'inventaire Ansible
        │
        ▼
Scénarios de déploiement
        │
        ▼
Installation et configuration des services
```

---

# Prérequis

## Contrôleur Ansible

- Ubuntu 24.04 LTS
- Ansible Core 2.16+
- Python 3
- Accès SSH aux serveurs cibles

## Serveurs cibles

Template standard :

- Ubuntu Server 24.04 LTS Minimal
- Compte `ansible_user`
- Sudo sans mot de passe
- Clé SSH bootstrap déployée
- OpenSSH Server
- Python 3
- Cloud-init
- Open VM Tools

---

# Arborescence du projet

```text
ansible/
├── inventories/
│   ├── hosts.yml
│   └── group_vars/
│       └── all.yml
│
├── playbooks/
│   ├── common/
│   │   ├── hostname.yml
│   │   └── update.yml
│   │
│   └── services/
│
├── roles/
│   ├── hostname/
│   ├── update/
│   └── ssh/
│
├── scenarios/
│   ├── bootstrap.yml
│   └── ubuntu_base.yml
│
└── vault/
    └── secret.yml
```

---

# Inventaire

Exemple :

```yaml
all:
  hosts:
    vm-zabbix-rec:
      ansible_host: 192.168.1.1
      hostname_vm: vm-zabbix
```

Variables globales :

```yaml
ansible_user: ansible
ansible_become: true
ansible_python_interpreter: python3
ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
```

---

# Rôles

## hostname

Configuration du nom de la machine.

Exemple :

```yaml
hostname_vm: vm-zabbix
```

---

## update

Mise à jour du système :

- apt update
- apt upgrade
- apt autoremove
- redémarrage automatique si nécessaire

---

## ssh

Gestion du durcissement SSH.

Déploiement :

```text
/etc/ssh/sshd_config.d/99-hardening.conf
```

Configuration :

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
```

---

# Scénarios

Les scénarios permettent d'enchaîner plusieurs playbooks afin de construire un serveur complet.

## Exemple

```yaml
---
- name: Update
  ansible.builtin.import_playbook: ../playbooks/common/update.yml

- name: Hostname
  ansible.builtin.import_playbook: ../playbooks/common/hostname.yml
```

Exécution :

```bash
ansible-playbook scenarios/ubuntu_base.yml
```

---

# Création d'une nouvelle machine

## 1. Déployer depuis le template VMware

Créer une nouvelle VM à partir du template :

```text
tpl-ubuntu2404-base-v1
```

---

## 2. Premier démarrage

Régénérer le Machine ID :

```bash
sudo systemd-machine-id-setup
```

Régénérer les clés SSH hôtes :

```bash
sudo dpkg-reconfigure openssh-server
```

---

## 3. Configurer le réseau

Modifier la configuration Netplan.

Validation :

```bash
sudo netplan try
```

Application :

```bash
sudo netplan apply
```

---

## 4. Mettre à jour l'inventaire

Ajouter la machine dans :

```text
inventories/hosts.yml
```

---

## 5. Vérifier la connectivité

```bash
ansible all -m ping
```

Résultat attendu :

```json
"ping": "pong"
```

---

## 6. Déployer les scénarios

Exemple :

```bash
ansible-playbook scenarios/ubuntu_base.yml
```

---

# Gestion des clés SSH

Le template contient déjà :

```text
/home/ansible/.ssh/authorized_keys
```

avec une clé publique bootstrap.

Cette configuration permet à Ansible de prendre immédiatement le contrôle de la machine après son clonage sans nécessiter d'authentification par mot de passe.

En cas de remplacement du serveur Ansible, les nouvelles clés pourront être déployées :

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ansible@serveur
```

ou via un futur rôle dédié.

---

# Roadmap

## Infrastructure

- [x] Template Ubuntu 24.04 LTS
- [x] Compte ansible
- [x] SSH par clé
- [x] Rôle hostname
- [x] Rôle update
- [ ] Rôle network
- [ ] Rôle storage

## Services

- [ ] FTP
- [ ] Docker
- [X] PostgreSQL
- [x] Zabbix Server
- [ ] Zabbix Agent
- [X] Nginx
- [ ] Node Exporter

---

# Philosophie

Les rôles doivent rester atomiques et ne réaliser qu'une seule fonction.

Les scénarios assemblent ces rôles afin de construire des serveurs complets adaptés aux besoins métiers tout en conservant un socle commun et reproductible.