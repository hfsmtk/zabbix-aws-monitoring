Projet Zabbix - Supervision Cloud sur AWS
École d'Ingénieurs
Cycle d'Ingénieur : « Génie Informatique »
2ACI-GB

1. INTRODUCTION
1.1 Contexte du projet
Dans un contexte où les infrastructures informatiques sont de plus en plus orientées vers le cloud, la supervision des systèmes devient indispensable afin de garantir leur disponibilité, leur performance et leur fiabilité. La mise en place d'outils de monitoring permet de détecter rapidement les anomalies et d'anticiper les incidents avant qu'ils n'impactent les utilisateurs.

1.2 Objectifs du projet
Les objectifs principaux de ce projet sont les suivants :

Déployer une infrastructure de supervision centralisée sur Amazon Web Services (AWS)
Mettre en place un serveur Zabbix conteneurisé à l'aide de Docker
Superviser un environnement hybride composé de machines Linux et Windows
Visualiser les métriques système en temps réel
Tester et valider le mécanisme d'alertes de Zabbix
1.3 Outils et technologies utilisés
Amazon Web Services (EC2, VPC, Security Groups, Internet Gateway)
Zabbix Server
Zabbix Agent (Linux & Windows)
Docker & Docker Compose
PostgreSQL 14
Ubuntu Server
Windows Server
2. ARCHITECTURE RÉSEAU
2.1 Vue d'ensemble
L'architecture réseau a été conçue en respectant les contraintes du AWS Learner Lab. Une VPC unique a été déployé dans la région us-east-1 (N. Virginia) avec un sous-réseau public afin de simplifier l'accès aux instances lors de la démonstration.

2.2 Configuration du VPC
Le VPC a été configuré avec une plage d'adresses IP privée permettant la communication interne entre les instances EC2.

2.3 Sous-réseau public
Toutes les instances EC2 ont été placées dans un sous-réseau public, associé à une Internet Gateway, permettant l'accès à Internet.

2.4 Groupes de sécurité
Les groupes de sécurité ont été configurés pour autoriser uniquement les ports nécessaires :

80 / 443 : Interface Web Zabbix
10050 / 10051 : Communication Zabbix
22 : SSH
3389 : RDP
ICMP : Ping
3. DÉPLOIEMENT DES INSTANCES EC2
L'infrastructure repose sur trois instances EC2 :

3.1 Serveur Zabbix
Type : t2.medium
Système : Ubuntu Server
Rôle : Hébergement du serveur Zabbix conteneurisé
3.2 Client Linux
Type : t3.medium
Système : Ubuntu
Rôle : Machine supervisée
3.3 Client Windows
Type : t3.medium
Système : Windows Server
Rôle : Machine supervisée
4. INSTALLATION ET CONFIGURATION DU SERVEUR ZABBIX
4.1 Connexion au serveur
La connexion au serveur Zabbix a été effectuée via SSH à l'aide d'une clé privée.

bash
ssh -i <clé-privée>.pem ubuntu@<IP-publique>
4.2 Installation de Docker et Docker Compose
Docker et Docker Compose ont été installés afin de déployer Zabbix sous forme de conteneurs.

bash
# Installation de Docker
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker

# Installation de Docker Compose
sudo apt install -y docker-compose
4.3 Fichier docker-compose.yml
Le serveur Zabbix, l'interface Web et la base de données MySQL 8.0 ont été déployés à l'aide d'un fichier docker-compose.yml.

yaml
version: '3.8'

services:
  postgres-server:
    image: postgres:14
    container_name: postgres-server
    restart: always
    environment:
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pwd
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - zabbix-net

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:6.0.26-alpine
    container_name: zabbix-server
    restart: always
    depends_on:
      - postgres-server
    environment:
      DB_SERVER_HOST: postgres-server
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pwd
      ZBX_CACHESIZE: 256M
    ports:
      - "10051:10051"
    networks:
      - zabbix-net

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:6.0.26-alpine
    container_name: zabbix-web
    restart: always
    depends_on:
      - postgres-server
      - zabbix-server
    environment:
      DB_SERVER_HOST: postgres-server
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pwd
      ZBX_SERVER_HOST: zabbix-server
      PHP_TZ: Africa/Casablanca
    ports:
      - "80:8080"
    networks:
      - zabbix-net

networks:
  zabbix-net:
    driver: bridge

volumes:
  postgres-data:

4.4 Démarrage des conteneurs
Les conteneurs ont été lancés avec succès et vérifiés à l'aide de la commande docker ps.

bash
docker-compose up -d
docker ps
4.5 Accès à l'interface Web
L'interface Zabbix est accessible via l'adresse IP publique du serveur :

http://<IP-publique>
Identifiants par défaut :

Username : Admin
Password : zabbix


5. CONFIGURATION DES CLIENTS (AGENTS)
5.1 Agent Zabbix sur Linux
L'agent Zabbix a été installé sur le client Linux, puis configuré pour communiquer avec l'adresse IP privée du serveur Zabbix.

5.1.1 Connexion au client
bash
ssh -i <clé-privée>.pem ubuntu@<IP-client-linux>
5.1.2 Installation de l'agent
bash
sudo apt update
sudo apt install -y zabbix-agent
5.1.3 Configuration de l'agent
bash
sudo nano /etc/zabbix/zabbix_agentd.conf
Modifier les paramètres suivants :

Server=<IP-privée-serveur-Zabbix>
ServerActive=<IP-privée-serveur-Zabbix>
Hostname=Client-Linux
5.1.4 Redémarrage du service
bash
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
sudo systemctl status zabbix-agent
5.2 Agent Zabbix sur Windows
L'agent Zabbix a été installé sur le client Windows via un installateur MSI et configuré avec l'IP du serveur.

5.2.1 Téléchargement et installation
Télécharger l'agent Zabbix depuis le site officiel
Exécuter l'installateur MSI
Configurer pendant l'installation :
Server : <IP-privée-serveur-Zabbix>
Hostname : Client-Windows
5.2.2 Vérification du service
Vérifier que le service Zabbix Agent est actif dans les Services Windows.

6. MONITORING ET TABLEAUX DE BORD
6.1 Ajout des hôtes
Les hôtes Linux et Windows ont été ajoutés dans l'interface Zabbix avec les templates appropriés :

Accéder à Configuration → Hosts
Cliquer sur Create host
Renseigner :
Host name : Client-Linux ou Client-Windows
Groups : Sélectionner ou créer un groupe
Interfaces : Ajouter l'adresse IP privée
Templates : Ajouter Template OS Linux ou Template OS Windows
6.2 Vérification du statut
Les deux hôtes apparaissent avec le statut ZBX vert, confirmant la bonne communication entre les agents et le serveur.

6.3 Visualisation des métriques
Zabbix permet de visualiser des graphiques pour :

Client Linux
Charge CPU
Utilisation mémoire
Espace disque
Trafic réseau
Nombre de processus
Client Windows
Charge CPU
Utilisation mémoire
Espace disque
Trafic réseau
Services système
7. TESTS ET VALIDATION
Les tests suivants ont été réalisés pour valider le fonctionnement de la solution :

Vérification de la connectivité entre le serveur et les agents
Collecte des métriques système (CPU, mémoire, disque, réseau)
Visualisation des graphiques en temps réel
Test du mécanisme d'alertes
Validation de la supervision hybride Linux/Windows
8. CONCLUSION
Ce projet a permis de déployer avec succès une solution complète de supervision cloud sur AWS à l'aide de Zabbix et Docker. La supervision d'un environnement hybride Linux et Windows a été validée, incluant la collecte de métriques et la gestion des alertes.

Points clés
Déploiement réussi d'une infrastructure Zabbix conteneurisée sur AWS
Supervision efficace d'environnements Linux et Windows
Visualisation en temps réel des métriques système
Architecture scalable et maintenable
Perspectives d'amélioration
Mise en place d'alertes avancées (email, SMS)
Configuration de tableaux de bord personnalisés
Ajout de la supervision d'applications et de services
Implémentation de la haute disponibilité
Automatisation du déploiement avec Terraform ou CloudFormation
Auteur : Mottaki Hafsa
Projet : ProjetZabbix
Date : 2024-2025

