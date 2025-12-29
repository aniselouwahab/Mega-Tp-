
# Déploiement Automatisé d'une Infrastructure HA & Sécurisée

## 1. Description du Projet

Ce projet consiste à transformer un déploiement manuel en une approche Infrastructure as Code (IaC). L'objectif est de livrer une infrastructure hétérogène (Ubuntu, RedHat, Windows Server) capable de tolérer des pannes, sécurisée par défaut et surveillée en temps réel via Zabbix.

## Structure du Dépôt

```bash
Mega-Tp-/
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

## Note sur l'Adressage Réseau (Choix de Configuration)

Bien que le Vagrantfile prévoie un réseau privé statique (192.168.56.x), j'ai choisi d'effectuer le déploiement sur l'interface NAT (eth0) fournie par VMware (192.168.159.x).

 Pourquoi ce choix ?

- Stabilité du Provisioning : L'interface NAT est celle utilisée par Vagrant pour le SSH. En utilisant cette même interface pour Ansible et Zabbix, on élimine les risques de problèmes de routage entre les différentes cartes réseaux virtuelles.

- Accès Internet Simplifié : Cela permet aux services (comme l'Agent Zabbix ou Nginx) de communiquer sur le même segment que celui qui télécharge les mises à jour, simplifiant ainsi la configuration du pare-feu (firewalld).

- Réalité de l'Environnement : VMware Workstation priorise souvent son propre DHCP NAT. Utiliser ces adresses garantit que l'infrastructure est immédiatement fonctionnelle dès le **vagrant up**, sans configuration manuelle des routes statiques sur l'hôte.

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

 Si la commande **sudo pcs status** affiche une erreur de type **unable to get cib**, c'est que les services de cluster doivent être synchronisés manuellement après le premier déploiement :

```bash
# Sur node01 et node02
sudo pcs cluster start --all
sudo pcs cluster enable --all
```
   

## 5. Explication des Choix Techniques

 Cette infrastructure repose sur une architecture n-tier hautement disponible, où chaque service est protégé contre les défaillances matérielles et logicielles.

### A. Mécanisme de Haute Disponibilité (La Redondance)

  Notre stratégie repose sur un cluster de type Actif/Passif piloté par la suite Linux HA :

   - Corosync (La Communication) : Il sert de couche de messagerie. Il permet aux nœuds de s'envoyer des "heartbeats" (battements de cœur) pour confirmer qu'ils sont vivants. Si node01 ne répond plus, node02 le sait en quelques millisecondes.

   - Pacemaker (Le Cerveau) : C'est le gestionnaire de ressources. C'est lui qui prend la décision de déplacer les services. Nous avons configuré des contraintes de colocation pour garantir que l'IP Virtuelle (VIP), le serveur Web Nginx et le partage Samba tournent toujours sur le même nœud.

   - La VIP (IP Flottante) : L'adresse 192.168.159.200 est assignée dynamiquement par Pacemaker. En cas de panne, elle est "arrachée" au nœud défaillant et remontée sur le nœud sain. Pour l'utilisateur final, l'interruption est quasi invisible (perte d'un seul ping).


### B. Sécurisation : Défense en Profondeur (Hardening)

La sécurité n'est pas une option mais un socle intégré via Ansible :

  - Isolation Réseau : Utilisation de firewalld avec une politique par défaut "drop". Seuls les flux critiques (VRRP pour le cluster, ports 80, 445, 10050) sont autorisés.

  - SSH Hardening : Désactivation de l'accès root et forçage de l'authentification par clé SSH sur les nœuds Linux pour prévenir les attaques par force brute.

  - Active Directory Hardening : Sur Windows, nous avons automatisé la configuration de politiques de mots de passe complexes et la désactivation de services non essentiels pour réduire la surface d'attaque du domaine.

### C. Supervision Proactive avec Zabbix

Le monitoring ne se contente pas de collecter des données, il sert d'outil d'aide à la décision :

  - Monitoring de Service : Nous surveillons l'état du port 80. Si Nginx crash mais que la VM reste allumée, Zabbix le détecte immédiatement.

  - Gestion des Triggers : Des seuils d'alerte (CPU > 80%, RAM > 90%) permettent d'anticiper une saturation du cluster avant que le basculement ne devienne critique.

  - Alerte Visuelle : Un tableau de bord centralisé permet à l'administrateur de visualiser instantanément l'état de santé global des 4 VMs sur un seul écran.


### D. Pourquoi cette architecture ?

   Le choix de RedHat/AlmaLinux pour les nœuds de production s'explique par sa stabilité et son support natif des outils de clustering professionnels (pcs). L'utilisation d'une VM Admin sous Ubuntu permet de séparer les responsabilités : la gestion (Ansible) et la surveillance (Zabbix) ne sont pas impactées si le cluster de production subit une charge intense.


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

