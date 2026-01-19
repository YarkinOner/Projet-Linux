# Projet Fil Rouge - Services Reseaux Linux

# 1. Infrastructure de base

### Création des Machines Virtuelle
* Pour server1 et server2 :
* Mémoire RAM : 2 Go
* CPU : 1–2
* Disque : 20 Go (VDI – Dynamically allocated)
* ISO : Ubuntu Server 22.04.5 LTS (Live Server)
* Adaptateur Réseau 1 : NAT
* Adaptateur Réseau 2 : Internal Network → lan1
* OpenSSH Server : installé
* Import SSH key : NON

### Installation de Ubuntu Server
Pendant l’installation :
* Réseau : Use wired connection
* Stockage : Use entire disk
* Installer OpenSSH Server : OUI
* Importer une identité SSH : NON

### Configuration Réseau de Server1
* Création du fichier Netplan :

```` sudo nano /etc/netplan/01-network.yaml ````
 * Contenu ajouté :

 ````
 network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses:
        - 192.168.10.10/24
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
````

* Correction des permissions :

```` sudo chmod 600 /etc/netplan/01-network.yaml ````
* Application de la configuration :

````sudo netplan apply````

* Vérification des interfaces :

```` ip a````
### Activation du Service SSH (Server1)

```` 
sudo systemctl start ssh
sudo systemctl enable ssh
systemctl status ssh
````
* État attendu : active (running)

### Installation de Server2
* Server2 a été installé exactement comme Server1.

### Configuration Réseau de Server2

* Création du fichier Netplan :

```` sudo nano /etc/netplan/01-network.yaml````

* Contenu ajouté :

````
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses:
        - 192.168.10.11/24
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
````

* Permissions :

```` sudo chmod 600 /etc/netplan/01-network.yaml````

* Application :

````sudo netplan apply````

### Tests de Connexion

* Ping Server1 → Server2

````ping 192.168.10.11````

* Ping Server2 → Server1

```` ping 192.168.10.10 ````

* SSH depuis Server1 vers Server2

```` ssh oner@192.168.10.11````

* Tout fonctionne : les deux serveurs peuvent communiquer via le réseau interne.

# 2. Sauvegarde et restauration

* Cette section décrit la mise en place d’un système complet de sauvegarde entre les deux serveurs de l’infrastructure :

* Server 1 : yarkin — 192.168.10.10

* Server 2 : oner — 192.168.10.11

* L’objectif est d’assurer une copie régulière et automatique des fichiers critiques du système, ainsi que la possibilité de les restaurer facilement.

### Structure de Sauvegarde
* Les sauvegardes du serveur yarkin sont envoyées vers le serveur oner dans le répertoire :

```` /backup/server1/ ````

* Les fichiers sauvegardés proviennent principalement du répertoire :

```` /etc/````
* Ce répertoire contient toutes les configurations essentielles du système (réseau, SSH, services…).

### Préparation du Serveur de Destination (oner)

* Sur le serveur oner, le répertoire de sauvegarde est créé :

````
sudo mkdir -p /backup/server1
sudo chmod 777 /backup/server1
````
* Ce dossier accueille les fichiers envoyés depuis yarkin.

### Sauvegarde Manuelle avec rsync (yarkin → oner)
* Depuis le serveur yarkin, une première sauvegarde est envoyée vers oner :

````
rsync -av /etc/ oner@192.168.10.11:/backup/server1/etc/ 
````

* -a : mode archive (préserve droits et permissions)
* -v : mode verbose
* /etc/ : répertoire source
* destination : /backup/server1/etc/ sur le serveur oner
* Après exécution, le serveur oner contient une copie complète de /etc.

### Vérification sur le Serveur de Destination (oner)

* Pour s’assurer du bon transfert :

```` ls /backup/server1/etc ````

* On doit y retrouver les fichiers suivants, par exemple :

(image)

### Sauvegarde Automatique (Cron) – Tous les jours à 02h00

* Une tâche cron a été créée sur le serveur yarkin afin d’automatiser la sauvegarde quotidienne.

* Ouverture du crontab :

```` crontab -e ````

* Ajout de la ligne suivante :

````
0 2 * * * rsync -av /etc/ oner@192.168.10.11:/backup/server1/etc/ >/dev/null 2>&1
````

#### Explication :

* 0 2 * * *** → exécution à 02:00 chaque jour
* Sauvegarde automatique de /etc
* Envoi vers oner
* Sortie silencieuse (>/dev/null 2>&1)
* Ainsi, la sauvegarde est désormais totalement automatisée.

### Procédure de Restauration

* En cas de problème sur le serveur yarkin, il est possible de restaurer les fichiers de configuration grâce à la commande suivante :

````
rsync -av oner@192.168.10.11:/backup/server1/etc/ /etc/
````
### Cette commande :

* récupère l’intégralité des fichiers sauvegardés
* les réinjecte dans le répertoire /etc du serveur yarkin

### Stratégie de Sauvegarde

### Résumé de la stratégie mise en place :

* Sauvegarde du répertoire /etc (fichiers critiques)
* Stockage externe sur le serveur oner
* Synchronisation quotidienne automatique à 02:00
* Restauration simple via rsync
* Tests de cohérence effectués après chaque sauvegarde

# 3. Services réseau
* Ce document présente la mise en place complète de trois services essentiels au sein de l’infrastructure :
* Un service Web sécurisé en HTTPS (Nginx + certificat SSL)
* Un serveur DNS interne (Bind9) permettant la résolution du domaine filrouge.lan
* Un service base de données MariaDB, sécurisé et accessible depuis le réseau interne
* Les services sont installés principalement sur le serveur :
* Serveur 1 : yarkin — 192.168.10.10
* Serveur 2 : oner — 192.168.10.11

## Service Web HTTPS (Nginx)

### Installation de Nginx

* Sur le serveur yarkin :

```` 
sudo apt update
sudo apt install nginx -y
````
* Vérification :

````
systemctl status nginx
````
### Génération du certificat SSL (self-signed)
 * Création du dossier SSL :
 ````
 sudo mkdir -p /etc/nginx/ssl
````

* Génération du certificat :

````
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/nginx/ssl/nginx.key \
-out /etc/nginx/ssl/nginx.crt
````

### Paramètres recommandés :

* Champ	: Valeur
* Country	: FR
* State	: Paris
* Locality : Paris
* Organization	: filrouge
* Organizational : Linux
* Common Name	: 192.168.10.10

### Configuration Nginx pour HTTPS

* Édition du fichier :

````
sudo nano /etc/nginx/sites-available/default
````

* Remplacer le contenu par :
````
server {
    listen 80;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name 192.168.10.10;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
````
* Redémarrer :
````
sudo nginx -t
sudo systemctl reload nginx
````
### Test HTTPS

* Sur yarkin :
````
curl -k https://localhost
````

* Sur oner :
````
curl -k https://192.168.10.10
````

* Résultat attendu : page HTML « Fil Rouge - yarkin (HTTPS OK) »

## Serveur DNS (Bind9)

### Installation

* Sur yarkin :
````
sudo apt install bind9 bind9utils bind9-doc -y
````

### Domaine choisi
* Le domaine interne :
````
filrouge.lan
````

### Configuration des zones DNS
* Éditer :
````
sudo nano /etc/bind/named.conf.local
````

* Ajouter :
````
zone "filrouge.lan" {
    type master;
    file "/etc/bind/db.filrouge.lan";
};
````

### Création du fichier de zone

* Copie du modèle :
````
sudo cp /etc/bind/db.local /etc/bind/db.filrouge.lan
sudo nano /etc/bind/db.filrouge.lan
````

* Contenu :
````
$TTL    604800
@       IN      SOA     ns1.filrouge.lan. admin.filrouge.lan. (
                              3         ; Serial
                              604800    ; Refresh
                              86400     ; Retry
                              2419200   ; Expire
                              604800 )  ; Negative Cache TTL

; Name servers
@       IN      NS      ns1.filrouge.lan.

; DNS Server
ns1     IN      A       192.168.10.10

; Hosts
yarkin  IN      A       192.168.10.10
oner    IN      A       192.168.10.11
www     IN      A       192.168.10.10
````
### Test DNS
````
sudo named-checkzone filrouge.lan /etc/bind/db.filrouge.lan
````

* Doit afficher OK.

### Configuration du résolveur (résolv.conf)
* Sur yarkin et oner :
````
Sur yarkin et oner :
````

* Mettre :
````
nameserver 192.168.10.10
````

### Tests finaux

* Sur les deux serveurs :
````
dig www.filrouge.lan
ping www.filrouge.lan
````

* Résultat attendu :
www.filrouge.lan
 → 192.168.10.10

 ## Service Base de Données — MariaDB

 ### Installation

 * Sur yarkin :
 ````
 sudo apt install mariadb-server mariadb-client -y
````

* Sécurisation :
````
sudo mysql_secure_installation
````

### Réponses :

* Root password : Yes
* Unix socket authentication : No
* Remove anonymous users : Yes
* Disable remote root login : Yes
* Remove test database : Yes

### Création de la base
* Connexion :
````
mysql -u root -p
````
* Créer la base :
````
CREATE DATABASE filrouge_db;
````
### Création des utilisateurs
* Local :
````
CREATE USER 'yarkin'@'localhost' IDENTIFIED BY 'MotDePasseFort!';
GRANT ALL PRIVILEGES ON filrouge_db.* TO 'yarkin'@'localhost';
````
* Distant (depuis oner) :
````
CREATE USER 'yarkin'@'192.168.10.11' IDENTIFIED BY 'MotDePasseFort!';
GRANT ALL PRIVILEGES ON filrouge_db.* TO 'yarkin'@'192.168.10.11';
FLUSH PRIVILEGES;
````

### Activation du réseau MariaDB

* Modifier :
````
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
````
* Changer :
````
bind-address = 127.0.0.1
````
* En :
````
 bind-address = 0.0.0.0
````
* Puis :
````
sudo systemctl restart mariadb
````
### Test depuis oner
* Installer le client :
````
sudo apt install mariadb-client -y
````
* Connexion :
````
mysql -u yarkin -p -h 192.168.10.10
````
* Dans MariaDB :
````
SHOW DATABASES;
````

* Base attendue : filrouge_db

# 4. Conteneurisation


* Dans cette partie, une partie de l’infrastructure a été conteneurisée à l’aide de Docker et Docker Compose afin de permettre un déploiement simple et automatisé.

## Création de l’arborescence du projet

```bash
mkdir -p ~/infra-compose
cd ~/infra-compose
````

## Création du contenu web

````
mkdir -p web
nano web/index.html
````
* Un fichier HTML simple est créé afin de servir de page web pour le serveur Nginx.

## Configuration du serveur Nginx

````
mkdir -p nginx
nano nginx/default.conf
````
* Ce fichier définit la configuration du serveur Nginx à l’intérieur du conteneur (port 80, racine web, index).

## Définition des services avec Docker Compose

````
nano docker-compose.yml
````

- Le fichier docker-compose.yml définit deux services:

web : serveur Nginx conteneurisé

db : base de données MariaDB conteneurisée

Il inclut également :

La publication des ports

Les variables d’environnement

Les volumes pour la persistance des données

## Déploiement de l’infrastructure

````
docker compose up -d
````
* Cette commande permet de démarrer l’ensemble de l’infrastructure en arrière-plan avec une seule commande.

## Vérification de l’état des conteneurs

````
docker ps
````
* Cette commande permet de vérifier que tous les conteneurs sont bien en cours d’exécution.


## Test du service web
````
curl http://localhost:8080
````
* Cette commande permet de vérifier que le serveur Nginx répond correctement.

[localhost](curl.png)

## Test de la base de données MariaDB

````
docker exec -it fr-db mariadb -u filrouge -pfilrougepass -e "SHOW DATABASES;"
`````
* Cette commande permet de vérifier l’accès à la base de données MariaDB depuis le conteneur et de confirmer la persistance des données.

[mariadb](docker.png)

# 5. Automatisation

## Objectif
Automatiser le déploiement de l’infrastructure afin qu’un nouvel arrivant puisse reconstruire l’environnement à partir des fichiers du projet, avec un minimum d’actions manuelles.

L’infrastructure conteneurisée (Partie 4) est pilotée via **Docker Compose** et un script de déploiement.

---

## 1) Accès au répertoire du projet

```bash
cd ~/infra-compose
ls
````
* Ces commandes permettent de se placer dans le répertoire contenant les fichiers de déploiement (docker-compose.yml, dossiers web/ et nginx/).

## Création du script de déploiement deploy.sh
````
nano deploy.sh
````
Contenu du script :

````
echo "Déploiement de l'infrastructure..."

docker compose pull
docker compose up -d

echo "Infrastructure prête."
````
* Rôle du script :

docker compose pull : récupère/met à jour les images nécessaires

docker compose up -d : démarre l’ensemble des services en arrière-plan

Les messages echo rendent l’exécution plus lisible

## Rendre le script exécutable
````
chmod +x deploy.sh
````
* Cette commande autorise l’exécution du script.

## Déploiement automatisé
````
./deploy.sh
````
* Cette commande permet de déployer l’infrastructure automatiquement.

## Vérification des conteneurs

````
docker ps
````
* Permet de vérifier que les services conteneurisés (ex : fr-web, fr-db) sont bien démarrés.

# 6. Surveillance

## Objectif
Mettre en place un système de surveillance afin de vérifier l’état des services, détecter rapidement les dysfonctionnements et faciliter le diagnostic en cas de problème.

La surveillance repose sur :
- Les **healthchecks Docker**
- La consultation des **logs**
- Un outil de supervision avec interface web (**Uptime Kuma**)

---

## Healthcheck du service Web (Nginx)

Un healthcheck est ajouté au conteneur Nginx afin de vérifier automatiquement que le service web répond correctement.

```yaml
healthcheck:
  test: ["CMD-SHELL", "wget -qO- http://localhost/ | grep -q 'Fil Rouge'"]
  interval: 10s
  timeout: 3s
  retries: 3
````

- Ce test vérifie que la page web est accessible et contient le texte attendu.
Si le test échoue, le conteneur est marqué comme unhealthy.

## Healthcheck du service Base de données (MariaDB)

- Un healthcheck est également configuré pour la base de données MariaDB.
````
healthcheck:
  test: ["CMD-SHELL", "mariadb-admin ping -uroot -p$${MARIADB_ROOT_PASSWORD} --silent"]
  interval: 10s
  timeout: 5s
  retries: 5
````

- Ce test permet de vérifier que le serveur MariaDB est opérationnel et accepte les connexions.

## Ajout d’un service de monitoring (Uptime Kuma)
````
uptime-kuma:
  image: louislam/uptime-kuma:latest
  container_name: fr-monitor
  ports:
    - "3001:3001"
  volumes:
    - kuma_data:/app/data
  restart: unless-stopped
````
- Uptime Kuma permet de surveiller l’état des services via des checks HTTP ou TCP.

## Déploiement de la supervision
````
docker compose up -d
````
- Cette commande permet de démarrer tous les services, y compris ceux de surveillance.

## Vérification de l’état des conteneurs
````
docker ps
````
- La colonne STATUS indique l’état de santé des conteneurs (healthy / unhealthy).

- Pour une vérification plus précise :
````
docker inspect --format='{{.Name}} => {{.State.Health.Status}}' fr-web fr-db
````

[kuma](kuma.png)

## Consultation des logs
- Les logs permettent d’analyser rapidement un problème en cas de dysfonctionnement.

- Logs de tous les services :
````
docker compose logs -f

docker logs -f fr-web
docker logs -f fr-db
````
## Accès à l’interface de supervision
- L’interface Uptime Kuma est accessible sur le port 3001 :
````
curl -I http://localhost:3001
````

- Depuis l’interface web, il est possible d’ajouter des moniteurs afin de surveiller :

Le service web (HTTP)

La base de données (TCP – port 3306)

- Grâce à cette mise en place, l’infrastructure dispose désormais :

- D’une surveillance automatique des services

- D’indicateurs de santé clairs

- D’un outil centralisé de supervision

- D’une meilleure réactivité en cas d’incident


Yarkin ONER


