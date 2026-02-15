# Guide d'Installation Privé - GLENZ Stack

**Document confidentiel - Usage interne**
Projet IntroSSI - UCAD ESP
Instructions détaillées d'installation pas à pas

---

## Table des matières

1. [Prérequis matériels et logiciels](#1-prérequis)
2. [Préparation du système](#2-préparation-du-système)
3. [Installation de Docker](#3-installation-de-docker)
4. [Déploiement du projet](#4-déploiement-du-projet)
5. [Configuration réseau](#5-configuration-réseau)
6. [Vérification et tests](#6-vérification-et-tests)
7. [Configuration Grafana](#7-configuration-grafana)
8. [Maintenance quotidienne](#8-maintenance-quotidienne)
9. [Dépannage](#9-dépannage)

---

## Processus d'Installation - Vue d'ensemble

```mermaid
flowchart TD
    Start([🚀 Début Installation])

    Start --> Prereq{Vérifier<br/>Prérequis}
    Prereq -->|OK| PrepSystem[📋 Préparer Système<br/>Ubuntu 22.04]
    Prereq -->|Manquant| InstallReq[Installer<br/>Prérequis]
    InstallReq --> PrepSystem

    PrepSystem --> InstallDocker[🐳 Installer Docker<br/>+ Docker Compose]
    InstallDocker --> ConfigSystem[⚙️ Configurer Système<br/>vm.max_map_count]

    ConfigSystem --> DeployProject[📦 Déployer Projet<br/>Clone/Upload]
    DeployProject --> ConfigNetwork[🌐 Config Réseau<br/>Port Mirroring]

    ConfigNetwork --> LaunchStack[🚀 Lancer Stack<br/>docker compose up]
    LaunchStack --> WaitServices[⏱️ Attendre Services<br/>3-5 minutes]

    WaitServices --> TestCapture{Test Capture<br/>Fonctionne?}
    TestCapture -->|✅ Oui| TestGrafana{Grafana<br/>Accessible?}
    TestCapture -->|❌ Non| Debug1[🐛 Debug Capture<br/>Interface, Permissions]
    Debug1 --> TestCapture

    TestGrafana -->|✅ Oui| TestData{Données dans<br/>Elasticsearch?}
    TestGrafana -->|❌ Non| Debug2[🐛 Debug Grafana<br/>Logs, Healthcheck]
    Debug2 --> TestGrafana

    TestData -->|✅ Oui| Success([✅ Installation<br/>Réussie!])
    TestData -->|❌ Non| Debug3[🐛 Debug Logstash<br/>Pipelines, Parsing]
    Debug3 --> TestData

    Success --> Maintenance[🔧 Maintenance]

    style Start fill:#e1f5fe
    style Success fill:#c8e6c9
    style Debug1 fill:#ffccbc
    style Debug2 fill:#ffccbc
    style Debug3 fill:#ffccbc
```

---

## 1. Prérequis

### 1.1 Matériel

**Configuration minimale**:
- CPU: 2 cœurs (Intel/AMD)
- RAM: 4 GB
- Disque: 50 GB d'espace libre
- Réseau: 1 interface Gigabit minimum

**Configuration recommandée**:
- CPU: 4+ cœurs
- RAM: 8 GB
- Disque: 200 GB SSD
- Réseau: 2 interfaces (management + capture)

### 1.2 Logiciels

- **OS**: Ubuntu Server 22.04 LTS (recommandé)
- **Docker**: Version 20.10+
- **Docker Compose**: Version 2.0+
- **Accès**: SSH, privilèges sudo

### 1.3 Réseau

- Accès Internet pour télécharger les images
- Possibilité de configurer port mirroring (SPAN)
- Adresse IP statique recommandée

---

## 2. Préparation du système

### 2.1 Installation fraîche d'Ubuntu

```bash
# 1. Télécharger Ubuntu Server 22.04 LTS
# https://ubuntu.com/download/server

# 2. Créer une clé USB bootable avec Rufus ou dd
dd if=ubuntu-22.04-live-server-amd64.iso of=/dev/sdX bs=4M status=progress

# 3. Installer Ubuntu
# - Choisir "Ubuntu Server" (minimal)
# - Configurer le réseau en IP statique
# - Créer un utilisateur (ex: labadmin)
# - Installer OpenSSH server
```

### 2.2 Mise à jour du système

```bash
# Se connecter en SSH ou console
ssh labadmin@<IP_SERVER>

# Mise à jour complète
sudo apt update
sudo apt upgrade -y
sudo apt dist-upgrade -y

# Redémarrer si kernel mis à jour
sudo reboot
```

### 2.3 Configuration du hostname

```bash
# Définir un hostname explicite
sudo hostnamectl set-hostname glenz-stack

# Vérifier
hostnamectl
```

### 2.4 Installation des outils nécessaires

```bash
# Installer les paquets de base
sudo apt install -y \
    curl \
    wget \
    git \
    jq \
    net-tools \
    htop \
    vim \
    tcpdump \
    wireshark-common

# Ajouter l'utilisateur aux groupes nécessaires
sudo usermod -aG wireshark $USER
```

---

## 3. Installation de Docker

### 3.1 Désinstaller anciennes versions

```bash
sudo apt remove docker docker-engine docker.io containerd runc
```

### 3.2 Installation via le script officiel

```bash
# Télécharger et exécuter le script d'installation
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Ajouter l'utilisateur au groupe docker
sudo usermod -aG docker $USER

# Appliquer les changements
newgrp docker

# Vérifier l'installation
docker --version
docker compose version
```

### 3.3 Configuration système pour Elasticsearch

```bash
# Augmenter vm.max_map_count (requis pour Elasticsearch)
sudo sysctl -w vm.max_map_count=262144

# Rendre permanent
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# Vérifier
sysctl vm.max_map_count
```

### 3.4 Test Docker

```bash
# Tester que Docker fonctionne
docker run hello-world

# Vérifier Docker Compose
docker compose version
```

---

## 4. Déploiement du projet

### 4.1 Récupération des fichiers

**Option A: Clone Git (si disponible)**
```bash
cd ~
git clone <REPO_URL> surveillance-reseau
cd surveillance-reseau
```

**Option B: Upload manuel**
```bash
# Sur votre machine locale
scp -r surveillance-reseau/ labadmin@<IP_SERVER>:~/

# Sur le serveur
cd ~/surveillance-reseau
```

### 4.2 Configuration de l'environnement

```bash
# Copier le fichier d'exemple
cp .env.example .env

# Éditer la configuration
nano .env
```

**Modifier ces valeurs**:
```bash
# Interface réseau à surveiller (utiliser: ip link show)
CAPTURE_INTERFACE=ens33

# Mémoire Elasticsearch (2g recommandé)
ES_JAVA_OPTS=-Xms2g -Xmx2g

# Mémoire Logstash (1g recommandé)
LOGSTASH_JAVA_OPTS=-Xms1g -Xmx1g

# Mot de passe Grafana
GRAFANA_PASSWORD=VotreMotDePasseSecurise123!

# Rétention PCAP (en jours)
PCAP_RETENTION_DAYS=7

# Configuration ring buffer PCAP
PCAP_MAX_SIZE=1000  # 1GB par fichier
PCAP_FILES=10       # 10 fichiers max
```

### 4.3 Vérification de la structure

```bash
# Vérifier la structure du projet
tree -L 2

# Devrait afficher:
# .
# ├── configs/
# │   ├── grafana/
# │   ├── logstash/
# │   ├── nginx/
# │   └── zeek/
# ├── data/
# ├── docs/
# ├── scripts/
# ├── docker-compose.yml
# └── .env
```

### 4.4 Création des répertoires de données

```bash
# Créer les répertoires nécessaires
mkdir -p data/{elasticsearch,grafana,logs/{zeek,ettercap},pcap}

# Vérifier les permissions
ls -la data/
```

---

## 5. Configuration réseau

### 5.1 Identifier l'interface de capture

```bash
# Lister les interfaces réseau
ip link show

# Exemple de sortie:
# 1: lo: <LOOPBACK,UP,LOWER_UP>
# 2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP>
# 3: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP>

# Utiliser l'interface connectée au réseau à surveiller
# Par défaut: ens33
```

### 5.2 Mode promiscuité (si nécessaire)

```bash
# Activer le mode promiscuité sur l'interface de capture
sudo ip link set ens33 promisc on

# Vérifier
ip link show ens33 | grep PROMISC

# Rendre permanent (ajouter à /etc/rc.local)
echo "ip link set ens33 promisc on" | sudo tee -a /etc/rc.local
```

### 5.3 Configuration du port mirroring (Switch)

**Pour un switch Cisco:**
```
# Configuration SPAN (Port Mirroring)
configure terminal
monitor session 1 source interface GigabitEthernet0/1 both
monitor session 1 destination interface GigabitEthernet0/24
end
write memory
```

**Pour un switch manageable générique:**
- Se connecter à l'interface web du switch
- Activer "Port Mirroring" ou "SPAN"
- Source: Port(s) à surveiller
- Destination: Port du serveur GLENZ

---

## 6. Vérification et tests

### 6.1 Démarrage de la stack

```bash
# Démarrer tous les services
docker compose up -d

# Vérifier que tous les containers démarrent
docker compose ps

# Devrait afficher:
# NAME                              STATUS
# surveillance-elasticsearch        Up (healthy)
# surveillance-grafana              Up (healthy)
# surveillance-zeek                 Up
# surveillance-logstash             Up
# surveillance-dumpcap              Up
# surveillance-ettercap             Up
# surveillance-nginx                Up
```

### 6.2 Vérification des logs

```bash
# Logs de tous les services
docker compose logs -f

# Logs d'un service spécifique
docker compose logs -f zeek
docker compose logs -f logstash
docker compose logs -f grafana

# Vérifier qu'il n'y a pas d'erreurs critiques
```

### 6.3 Tests automatiques

```bash
# Exécuter le script de tests
bash scripts/tests.sh

# Résultat attendu:
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# ✓ TOUS LES TESTS SONT PASSÉS !
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 6.4 Vérification Elasticsearch

```bash
# Santé du cluster
curl http://localhost:9200/_cluster/health?pretty

# Lister les index
curl http://localhost:9200/_cat/indices?v

# Devrait afficher:
# health status index              pri rep docs.count
# yellow open   zeek-conn-2024.02.15  1   1       5000
# yellow open   zeek-dns-2024.02.15   1   1       3000
# yellow open   zeek-http-2024.02.15  1   1       1500

# Compter les documents
curl http://localhost:9200/zeek-conn-*/_count?pretty
```

### 6.5 Vérification Zeek

```bash
# Vérifier que Zeek génère des logs
ls -lh data/logs/zeek/current/

# Devrait afficher:
# conn.log   - Connexions réseau
# dns.log    - Requêtes DNS
# http.log   - Trafic HTTP
# notice.log - Alertes

# Voir le contenu (sans les commentaires)
grep -v "^#" data/logs/zeek/current/conn.log | head -5
```

### 6.6 Vérification PCAP

```bash
# Vérifier que les PCAP sont créés
ls -lh data/pcap/$(date +%Y-%m-%d)/

# Devrait afficher des fichiers .pcap
# -rw-r--r-- 1 root root 50M Feb 15 10:00 capture_00001.pcap
# -rw-r--r-- 1 root root 25M Feb 15 10:30 capture_00002.pcap
```

---

## 7. Configuration Grafana

### 7.1 Accès initial

```bash
# Ouvrir Grafana dans le navigateur
http://<IP_SERVER>:3000

# Identifiants par défaut:
# User: admin
# Pass: admin (ou celui défini dans .env)
```

### 7.2 Vérification de la datasource

1. Aller dans **Configuration** → **Data Sources**
2. Vérifier que "Elasticsearch-Zeek" est présent
3. Cliquer sur "Test" → Devrait afficher "Data source is working"

**Si la datasource n'existe pas:**
```bash
# Créer manuellement via l'API
curl -X POST http://admin:admin@localhost:3000/api/datasources \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Elasticsearch-Zeek",
    "type": "elasticsearch",
    "url": "http://elasticsearch:9200",
    "access": "proxy",
    "database": "zeek-*",
    "jsonData": {
      "timeField": "@timestamp",
      "esVersion": "8.0.0",
      "interval": "Daily"
    }
  }'
```

### 7.3 Création d'un dashboard de test

1. Aller dans **Dashboards** → **New Dashboard**
2. Cliquer sur **Add visualization**
3. Sélectionner "Elasticsearch-Zeek"
4. Configurer:
   - **Query**: `log_type:zeek AND event_type:connection`
   - **Metric**: Count
   - **Time Field**: @timestamp
5. Cliquer sur **Apply**

### 7.4 Importer les dashboards pré-configurés (si disponibles)

```bash
# Depuis le répertoire du projet
# Les dashboards seront dans configs/grafana/dashboards/

# Ils seront automatiquement chargés au démarrage
```

**Dashboards disponibles:**
- Zeek Network Overview
- DNS Analysis
- HTTP Traffic Analysis
- Security Alerts
- MITM Detection

---

## 8. Maintenance quotidienne

### 8.1 Vérification des services

```bash
# Vérifier l'état des containers
docker compose ps

# Vérifier l'utilisation des ressources
docker stats

# Vérifier les logs pour les erreurs
docker compose logs --tail=100 | grep -i error
```

### 8.2 Nettoyage des données

```bash
# Vérifier l'espace disque
df -h

# Taille des données
du -sh data/*

# Nettoyer les anciens index Elasticsearch (>30 jours)
curl -X DELETE "http://localhost:9200/zeek-*-$(date -d '30 days ago' +%Y.%m.%d)"

# Nettoyer les anciens PCAP (automatique via script)
find data/pcap -type d -mtime +7 -exec rm -rf {} \;
```

### 8.3 Backup

```bash
# Backup de la configuration
tar -czf backup-config-$(date +%Y%m%d).tar.gz configs/

# Backup des dashboards Grafana
curl -X GET http://admin:admin@localhost:3000/api/search > dashboards-backup.json

# Backup Elasticsearch (snapshot)
curl -X PUT "http://localhost:9200/_snapshot/my_backup" -H 'Content-Type: application/json' -d'
{
  "type": "fs",
  "settings": {
    "location": "/data/backups"
  }
}'
```

### 8.4 Mise à jour

```bash
# Mettre à jour les images Docker
docker compose pull

# Redémarrer avec les nouvelles images
docker compose up -d

# Vérifier que tout fonctionne
bash scripts/tests.sh
```

---

## 9. Dépannage

### 9.1 Elasticsearch ne démarre pas

**Symptôme**: Container `surveillance-elasticsearch` en état "Restarting"

**Solutions**:
```bash
# Vérifier les logs
docker compose logs elasticsearch

# Erreur commune: "vm.max_map_count too low"
sudo sysctl -w vm.max_map_count=262144

# Vérifier les permissions
sudo chown -R 1000:1000 data/elasticsearch

# Redémarrer
docker compose restart elasticsearch
```

### 9.2 Zeek ne capture pas de trafic

**Symptôme**: Fichiers logs Zeek vides ou inexistants

**Solutions**:
```bash
# Vérifier que l'interface existe
ip link show ens33

# Vérifier les logs Zeek
docker compose logs zeek

# Vérifier le mode promiscuité
ip link show ens33 | grep PROMISC

# Activer manuellement
sudo ip link set ens33 promisc on

# Redémarrer Zeek
docker compose restart zeek
```

### 9.3 Logstash ne parse pas les logs

**Symptôme**: Pas de documents dans les index Elasticsearch

**Solutions**:
```bash
# Vérifier les logs Logstash
docker compose logs logstash | grep -i error

# Vérifier les pipelines
ls -la configs/logstash/pipelines/

# Vérifier que les logs Zeek existent
ls -la data/logs/zeek/current/

# Tester le parsing manuellement
docker compose exec logstash logstash -f /usr/share/logstash/pipeline/01-zeek-conn.conf --config.test_and_exit

# Redémarrer Logstash
docker compose restart logstash
```

### 9.4 Grafana ne se connecte pas à Elasticsearch

**Symptôme**: Erreur "Bad Gateway" ou "Data source is not working"

**Solutions**:
```bash
# Vérifier qu'Elasticsearch est accessible
curl http://localhost:9200/_cluster/health

# Vérifier la configuration de la datasource
curl http://admin:admin@localhost:3000/api/datasources

# Recréer la datasource
curl -X POST http://admin:admin@localhost:3000/api/datasources \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Elasticsearch-Zeek",
    "type": "elasticsearch",
    "url": "http://elasticsearch:9200",
    "access": "proxy",
    "database": "zeek-*",
    "jsonData": {
      "timeField": "@timestamp",
      "esVersion": "8.0.0"
    }
  }'

# Redémarrer Grafana
docker compose restart grafana
```

### 9.5 Problèmes de performance

**Symptôme**: Système lent, containers qui redémarrent

**Solutions**:
```bash
# Vérifier l'utilisation des ressources
docker stats

# Réduire la mémoire Elasticsearch (dans .env)
ES_JAVA_OPTS=-Xms1g -Xmx1g

# Réduire la mémoire Logstash
LOGSTASH_JAVA_OPTS=-Xms512m -Xmx512m

# Augmenter le swap (si nécessaire)
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Redémarrer les services
docker compose down
docker compose up -d
```

### 9.6 Logs utiles pour le debug

```bash
# Tout afficher
docker compose logs -f

# Elasticsearch
docker compose logs -f elasticsearch | grep -i error

# Logstash
docker compose logs -f logstash | grep -i "error\|warn"

# Zeek
docker compose logs -f zeek

# Grafana
docker compose logs -f grafana | grep -i error

# Dumpcap
docker compose logs -f dumpcap
```

---

## Contacts et Support

**Projet**: GLENZ Stack - UCAD ESP
**Email**: support@example.com
**Documentation**: `/docs/`

---

**Version**: 3.0 - GLENZ Stack
**Date**: Février 2024
**Auteur**: Département Génie Informatique - UCAD ESP
