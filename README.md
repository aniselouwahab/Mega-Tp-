
# Déploiement Automatisé d'une Infrastructure HA & Sécurisée

## 1. Description du Projet

Ce projet consiste à transformer un déploiement manuel en une approche Infrastructure as Code (IaC). L'objectif est de livrer une infrastructure hétérogène (Ubuntu, RedHat, Windows Server) capable de tolérer des pannes, sécurisée par défaut et surveillée en temps réel via Zabbix.

## Structure du Dépôt

```bash
TP_HA_DevOps/
├── Vagrantfile
├── hosts.ini
├── ansible.cfg
├── site.yml             
├── README.md               
└── ansible/                                         
    ├── cluster_ha.yml      
    ├── security_linux.yml  
    ├── windows_mission.yml  
    └── zabbix_agent.yml
```


## 2. Schéma d'Architecture Réseau

L'infrastructure est composée de 4 machines virtuelles interconnectées

![image alt](https://github.com/aniselouwahab/Mega-Tp-/blob/9538ae729755b805b45ac10928453d94ad9eac3a/Capture%20d%E2%80%99%C3%A9cran%202025-12-28%20204040.png)

## 3. Guide d'Installation et Commandes

### A. Prérequis
 -Vagrant et VirtualBox/vmware installés sur la machine hôte.

 -Une clé SSH configurée pour l'accès aux machines Linux.

## B. Récupération du projet

Ouvrez un terminal et clonez ce dépôt pour récupérer l'intégralité du code (Vagrantfile et Ansible):

```bash
git clone https://github.com/aniselouwahab/Mega-Tp-.git

cd Mega-Tp-
```

## C. Lancement de l'infrastructure

Déployez automatiquement les 4 machines virtuelles (Admin, Node01, Node02, WinSrv):

```bash
# Cette commande crée les VMs et configure le réseau selon le Vagrantfile
vagrant up
```

## 4. Configuration automatisée via Ansible

Une fois les machines prêtes, connectez-vous à la machine de contrôle pour lancer les missions de configuration (HA, Sécurité, AD, Zabbix):

```bash
# Connexion au nœud de contrôle Ubuntu
vagrant ssh admin

# Exécution du déploiement global depuis le dossier partagé
cd /vagrant
ansible-playbook -i hosts.ini site.yml
```

## NB:

    Si la commande **sudo pcs status** affiche une erreur de type unable to get cib, c'est que les services de cluster doivent être synchronisés manuellement après le premier déploiement :

```bash
# Sur node01 et node02
sudo pcs cluster start --all
sudo pcs cluster enable --all
```
   

## 5. Explication des Choix Techniques

  - Haute Disponibilité : Utilisation de Pacemaker et Corosync pour gérer le basculement automatique des services (Nginx/Samba) et de l'IP flottante (VIP).

  - Sécurisation : Application d'un "Hardening" strict : pare-feu firewalld, désactivation du root SSH sur Linux, et politiques de mots de passe complexes sur Windows AD.

  - Monitoring : Centralisation sur Zabbix Server (Admin) avec des agents sur chaque nœud pour un suivi en temps réel des performances CPU/RAM et de la disponibilité des services.



## 6. Preuves de Fonctionnement(Screenshots)

### Mission 1 : Haute Disponibilité (HA)

Le cluster Pacemaker/Corosync est opérationnel. L'IP flottante (VIP), Nginx et Samba sont configurés en groupe pour basculer ensemble.

 - État du Cluster : La commande **sudo pcs status** montre tous les nœuds "Online" et les ressources démarrées.

![image alt](https://github.com/aniselouwahab/Mega-Tp-/blob/9538ae729755b805b45ac10928453d94ad9eac3a/Capture%20d%E2%80%99%C3%A9cran%202025-12-28%20140021.png)


- Accès Web via VIP : La Test Page AlmaLinux s'affiche via l'adresse **192.168.159.200**

![image alt](https://github.com/aniselouwahab/Mega-Tp-/blob/9538ae729755b805b45ac10928453d94ad9eac3a/Capture%20d%E2%80%99%C3%A9cran%202025-12-27%20002924.png)


### Mission 2 : Sécurisation & Hardening

Le pare-feu firewalld est actif et ne laisse passer que les flux nécessaires (SSH, HTTP, Cluster, Zabbix)

 - Configuration Agent : L'agent Zabbix 2 communique de manière sécurisée avec le serveur 

 ![image alt](https://github.com/aniselouwahab/Mega-Tp-/blob/44b1a8e8640452ccda8429ec7df58b5970f3c884/Capture%20d%E2%80%99%C3%A9cran%202025-12-28%20194314.png)

### Mission 3 : Windows Server & Active Directory

Rôles AD DS : Le gestionnaire de serveur montre que les services de domaine Active Directory sont installés et configurés.
 ![image alt](https://github.com/aniselouwahab/Mega-Tp-/blob/1aa896fcf14c39780ac9d9bd48f1752002f68039/Capture%20d%E2%80%99%C3%A9cran%202025-12-28%20193543.png)
 
### Mission 4 : Supervision avec Zabbix

Le Dashboard personnalisé sur la VM Admin affiche la santé de tout le parc.

- Métriques CPU/RAM : Visualisation des graphiques de performance pour Node01 et Node02

![image alt](https://github.com/aniselouwahab/Mega-Tp-/blob/9538ae729755b805b45ac10928453d94ad9eac3a/Capture%20d%E2%80%99%C3%A9cran%202025-12-27%20002551.png)


- Disponibilité Nginx : Statut "UP" du service Web vérifié par l'agent.

![image alt](https://github.com/aniselouwahab/Mega-Tp-/blob/9538ae729755b805b45ac10928453d94ad9eac3a/Capture%20d%E2%80%99%C3%A9cran%202025-12-27%20002621.png)

- Alerte Visuelle :  Alerte rouge si un nœud ou le service Web ne répond plus.

![image alt](https://github.com/aniselouwahab/Mega-Tp-/blob/9538ae729755b805b45ac10928453d94ad9eac3a/Capture%20d%E2%80%99%C3%A9cran%202025-12-27%20002606.png)


- Lien Agent-Serveur : Capture d’écran 2025-12-28 194243.png (Preuve de config agent).

![image alt](https://github.com/aniselouwahab/Mega-Tp-/blob/9538ae729755b805b45ac10928453d94ad9eac3a/Capture%20d%E2%80%99%C3%A9cran%202025-12-28%20194243.png)
## License

[MIT](https://choosealicense.com/licenses/mit/)

