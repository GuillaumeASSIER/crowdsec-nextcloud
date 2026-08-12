# 🔒 CrowdSec + Nextcloud

Une stack Docker Compose complète pour héberger **Nextcloud de manière sécurisée** avec **CrowdSec** comme système de protection contre les attaques.

Ce projet implémente une architecture de sécurité robuste avec :
- **Nextcloud** : Stockage cloud auto-hébergé
- **OpenResty** : Reverse proxy Nginx avec intégration CrowdSec
- **CrowdSec** : Système de détection et de blocage des menaces
- **Certbot** : Gestion automatique des certificats SSL/TLS
- **MariaDB** : Base de données relationnelle
- **Redis** : Cache et sessions

## 📖 Documentation

### 📚 Liens utiles

- **[Article en français](https://www.50-nuances-octets.fr/posts/securiser-une-installation-nextcloud-avec-crowdsec/)**
- **[Article en anglais](https://www.50-nuances-octets.fr/en/posts/securing-a-nextcloud-installation-with-crowdsec/)**

- **[CrowdSec Documentation](https://docs.crowdsec.net/)**
- **[Nextcloud Documentation](https://docs.nextcloud.com/)**

### 🛠️ Fichiers de configuration

| Fichier | Description |
|---------|-------------|
| `nextcloud/compose.yml` | Configuration Docker Compose principale |
| `nextcloud/conf.d/nginx.conf` | Configuration Nginx/OpenResty |
| `nextcloud/conf.d/crowdsec_openresty.conf` | Intégration CrowdSec avec OpenResty |
| `nextcloud/crowdsec/acquis.yml` | Fichiers de log à monitorer |
| `nextcloud/crowdsec/crowdsec-openresty-bouncer.conf` | Configuration du bouncer |
| `AGENTS.md` | Détail des agents CrowdSec et scénarios |

## 🚀 Démarrage rapide

### Prérequis

- Docker & Docker Compose (v2.0+)
- Domaine DNS pointant vers votre serveur
- Accès root/sudo sur le serveur

### 1️⃣ Cloner le repository

```bash
git clone https://github.com/GuillaumeASSIER/crowdsec-nextcloud.git
cd crowdsec-nextcloud/nextcloud
```

### 2️⃣ Configurer les variables d'environnement

```bash
cp .env.example .env
nano .env  # Éditer avec vos valeurs
```

**Variables obligatoires :**

```bash
HOSTNAME=nextcloud.example.com          # Votre domaine
MYSQL_ROOT_PASSWORD=<mot_de_passe>     # Sécurisé!
MYSQL_PASSWORD=<mot_de_passe>          # Sécurisé!
NEXTCLOUD_ADMIN_USER=admin              # Admin Nextcloud
NEXTCLOUD_ADMIN_PASSWORD=<mot_de_passe> # Sécurisé!
```

### 3️⃣ Lancer les services

```bash
# Démarrer tous les services
docker-compose up -d

# Vérifier le statut
docker-compose ps

# Voir les logs en direct
docker-compose logs -f app
```

### 4️⃣ Initialiser Nextcloud

Accédez à `https://nextcloud.example.com` et terminez la configuration initiale.

### 5️⃣ Obtenir un certificat SSL

```bash
# CertBot s'exécute en tant que service
docker-compose exec certbot certbot certonly \
  --webroot -w /var/www/certbot \
  -d nextcloud.example.com \
  -m votre.email@example.com \
  --agree-tos --non-interactive
```

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────┐
│           Internet (Attaquants)                 │
└────────────────────┬────────────────────────────┘
                     │ HTTP/HTTPS
                     ▼
         ┌───────────────────────┐
         │   OpenResty (Nginx)   │ ◄─ Port 80/443
         │  + CrowdSec Bouncer   │
         └────────────┬──────────┘
                      │
      ┌───────────────┼───────────────┐
      │               │               │
      ▼               ▼               ▼
 ┌─────────┐    ┌──────────┐   ┌──────────┐
 │ Nextcloud│   │ CrowdSec │   │  MariaDB │
 │  (App)   │   │(Detection)│   │(Database)│
 └─────────┘    └──────────┘   └──────────┘
      │               │
      └───────────────┴─────────────┐
                      │             │
                      ▼             ▼
                  ┌────────────┐  ┌────────┐
                  │   Redis    │  │ Logs   │
                  │  (Cache)   │  │(Nginx) │
                  └────────────┘  └────────┘
```

**Flux de sécurité :**
1. Les requêtes arrivent sur OpenResty
2. CrowdSec analyse les logs en temps réel
3. Les IPs malveillantes sont bloquées par le bouncer
4. Seules les requêtes légitimes arrivent à Nextcloud

## 🛡️ Sécurité - Agents CrowdSec

Voir [AGENTS.md](./AGENTS.md) pour la documentation complète des agents et scénarios.

**Agents principaux utilisés :**
- `crowdsecurity/linux` - Détection d'intrusions système
- `crowdsecurity/sshd` - Brute force SSH
- `crowdsecurity/nginx` - Attaques web communes
- `crowdsecurity/http-cve` - Exploitation de CVE HTTP
- `crowdsecurity/base-http-scenarios` - Scénarios HTTP basiques
- `crowdsecurity/nextcloud` - Spécifique à Nextcloud
- Scénario personnalisé : `nextcloud-bf` - Brute force Nextcloud

## 📊 Monitoring

### Vérifier l'état de CrowdSec

```bash
# Accéder au conteneur CrowdSec
docker-compose exec crowdsec bash

# Voir les alertes bloquées
cscli alerts list

# Voir les décisions actives
cscli decisions list

# Voir les machines enregistrées
cscli machines list

# Voir les bouncers enregistrés
cscli bouncers list

# Logs en temps réel
cscli hub list
```

### Logs Nextcloud

```bash
# Logs applicatifs
docker-compose exec app tail -f /var/log/nextcloud.log

# Logs Nginx/OpenResty
docker-compose logs -f openresty

# Logs CrowdSec
docker-compose logs -f crowdsec

# Logs de la base de données
docker-compose logs -f db
```
