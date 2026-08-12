# 🤖 Agents et Scénarios CrowdSec

Ce document détaille les agents CrowdSec utilisés dans cette stack pour sécuriser Nextcloud.

## 📋 Vue d'ensemble

CrowdSec fonctionne avec une architecture modulaire :
- **Parsers** : Analysent et structurent les logs bruts
- **Enrichers** : Enrichissent les données avec contexte supplémentaire (géolocalisation, ASN, etc.)
- **Scenarios** : Détectent les patterns malveillants basés sur les données enrichies
- **Collections** : Groupes de parsers, enrichers et scénarios prêts à l'emploi

La stack utilise **7 collections principales** pour détecter et bloquer les menaces en temps réel.

## 🔧 Collections installées

Ces 7 collections forment la base de la détection de menaces :

### 1. `crowdsecurity/linux`
**Détection d'intrusions système Linux**
- Analyse `/var/log/auth.log` et `/var/log/syslog`
- Détecte : brute force, escalade de privilèges, abus sudo
- Protection SSH et système

### 2. `crowdsecurity/sshd`
**Sécurisation spécifique du service SSH**
- Logs sshd avec détection fine
- Détecte : brute force SSH, brute force lents, utilisateurs invalides
- Protection port 22

### 3. `crowdsecurity/nginx`
**Détection des attaques web couantes**
- Analyse logs Nginx/OpenResty en temps réel
- Détecte : traversée de répertoires, SQL injection, XSS, scans, fichiers sensibles
- Protection couche HTTP/HTTPS

### 4. `crowdsecurity/http-cve`
**Exploitation de CVE HTTP connues**
- Signatures de vulnérabilités : Log4Shell, Atlassian, Laravel, etc.
- Mise à jour automatique des signatures
- Protection exploits zéro-day

### 5. `crowdsecurity/base-http-scenarios`
**Scénarios HTTP basiques et génériques**
- Détecte : énumération (404), DoS (50x), scans de répertoires
- Baseline de protection HTTP
- Détection des crawlers malveillants

### 6. `crowdsecurity/nextcloud`
**Détection spécifique aux menaces Nextcloud**
- Monitore `/nextcloud/data/nextcloud.log`
- Détecte : brute force authentification, énumération de comptes
- Protection des credentials Nextcloud

### 7. `nextcloud-bf` (Scénario personnalisé)
**Brute force Nextcloud spécialisé**
- Agrège tentatives échouées par IP
- Génère alertes après N tentatives en T minutes
- Blocage automatique via OpenResty bouncer

---

## 🔄 Flux de détection

```
┌─────────────────────────────┐
│     1. Collection du log     │
│   /var/log/nginx/access.log │
│   /var/log/auth.log         │
│  /nextcloud/data/nextcloud.log │
└────────────┬────────────────┘
             │
             ▼
     ┌───────────────┐
     │  2. Parsers   │
     │ (structuration)│
     └───────┬───────┘
             │
             ▼
    ┌────────────────┐
    │  3. Enrichers  │
    │  (Géolocation) │
    │   (ASN, etc)   │
    └────────┬───────┘
             │
             ▼
   ┌──────────────────┐
   │  4. Scénarios    │
   │  (Détection)     │
   │ - bruteforce     │
   │ - scan           │
   │ - exploit        │
   └──────────┬───────┘
              │
              ▼
   ┌──────────────────────┐
   │  5. Décision         │
   │  (Ban, Alert, etc)   │
   └──────────┬───────────┘
              │
              ▼
   ┌──────────────────────┐
   │  6. Bouncer OpenResty│
   │  (Blocage HTTP)      │
   └──────────────────────┘
```

---

## ⚙️ Configuration des agents

### Modifier les paramètres de détection

Les scénarios peuvent être ajustés directement dans le conteneur ou via volumes :

```bash
# Accéder au conteneur CrowdSec
docker-compose exec crowdsec bash

# Éditer un scénario
vi /etc/crowdsec/scenarios/crowdsecurity-sshd.yaml
```

### Paramètres courants d'un scénario

```yaml
# Nombre d'événements avant alerte
threshold: 5

# Fenêtre de temps (groupement)
window: 1h

# Durée du ban
duration: 4h

# Grouper par champ (généralement IP source)
group_by: "{{.Source.IP}}"
```

---

## 📊 Monitoring des agents

### Voir les alertes CrowdSec

```bash
docker-compose exec crowdsec cscli alerts list
```

**Affiche** : Les alertes générées par les scénarios actifs.

### Voir les décisions (bans actifs)

```bash
docker-compose exec crowdsec cscli decisions list
```

**Affiche** : Les IPs actuellement bannies et la durée.

### Voir les scénarios actifs

```bash
docker-compose exec crowdsec cscli scenarios list
```

### Voir les collections installées

```bash
docker-compose exec crowdsec cscli hub list
```

### Voir les bouncers connectés

```bash
docker-compose exec crowdsec cscli bouncers list
```

**Doit afficher** : Le bouncer OpenResty comme connecté.

### Voir les logs CrowdSec

```bash
docker-compose logs -f crowdsec
```

---

## 🧪 Tester la détection

### Tester la détection SSH

```bash
# Depuis votre machine locale, tentez plusieurs connexions SSH échouées
for i in {1..10}; do
  ssh invalid@nextcloud.example.com 2>/dev/null || true
done

# Vérifier que l'IP est bannie
docker-compose exec crowdsec cscli decisions list
```

### Tester la détection HTTP

```bash
# Tenter un accès à un chemin sensible
curl -v http://localhost/.git/config
curl -v http://localhost/../../../etc/passwd

# Vérifier les alertes
docker-compose exec crowdsec cscli alerts list
```

### Tester le brute force Nextcloud

```bash
# Effectuer plusieurs tentatives de login échouées
for i in {1..5}; do
  curl -X POST https://nextcloud.example.com/index.php/login \
    -d "user=admin&password=wrong" 2>/dev/null
done

# Vérifier le ban
docker-compose exec crowdsec cscli decisions list
```

---

## 🔓 Débloquer une IP bannie

```bash
# Lister les décisions
docker-compose exec crowdsec cscli decisions list

# Supprimer une décision par ID
docker-compose exec crowdsec cscli decisions delete --id <ID>

# Ou par IP
docker-compose exec crowdsec cscli decisions delete --ip 192.0.2.42
```

---

## 📈 Métriques et statistiques

### Voir les parseurs actifs

```bash
docker-compose exec crowdsec cscli parsers list
```

### Voir les scénarios actifs

```bash
docker-compose exec crowdsec cscli scenarios list
```

### Voir les collections installées

```bash
docker-compose exec crowdsec cscli hub list
```

### Voir les bouncers connectés

```bash
docker-compose exec crowdsec cscli bouncers list
```

---

## 🚀 Ajouter de nouveaux agents

### Installation d'une nouvelle collection

```bash
# Accéder au conteneur
docker-compose exec crowdsec bash

# Ajouter une collection
cscli hub install collections/crowdsecurity/wordpress

# Recharger CrowdSec
systemctl restart crowdsec
```

### Ajouter via compose.yml

Modifier le service `crowdsec` dans `compose.yml` :

```yaml
crowdsec:
  environment:
    COLLECTIONS: "crowdsecurity/linux crowdsecurity/nginx crowdsecurity/wordpress"
```

---

## ⚠️ Considérations de sécurité

### Faux positifs

- Les scénarios peuvent détecter des comportements légitimes (notamment les scans)
- Ajuster les paramètres de `threshold` et `window`
- Monitorer régulièrement les alertes
- Whitelister les IPs de confiance

### Whitelisting

```bash
# Accéder au conteneur
docker-compose exec crowdsec bash

# Éditer le fichier de whitelist
vi /etc/crowdsec/bouncers/crowdsec-openresty-bouncer.conf

# Ajouter :
whitelist_ips:
  - 10.0.0.0/8
  - 172.16.0.0/12
```

### Performance

- CrowdSec consomme peu de ressources (~100-200 MB RAM)
- L'analyse des logs se fait en temps réel
- Monitorer avec : `docker stats crowdsec`

---

## 📚 Ressources supplémentaires

- [CrowdSec Hub](https://hub.crowdsec.net/) - Repository officiel des collections
- [CrowdSec Parsers](https://docs.crowdsec.net/docs/parsers/)
- [CrowdSec Scenarios](https://docs.crowdsec.net/docs/scenarios/)
- [OpenResty Bouncer](https://github.com/crowdsecurity/cs-openresty-bouncer)

---

## 🤔 FAQ

### Comment savoir si CrowdSec fonctionne ?

```bash
# Vérifier que le service tourne
docker-compose ps crowdsec

# Vérifier les logs
docker-compose logs crowdsec | tail -50

# Tester une connexion SSH échouée
ssh invalid@localhost
docker-compose exec crowdsec cscli alerts list
```

### Pourquoi je suis bloqué par CrowdSec ?

```bash
# Vérifier votre IP
curl ifconfig.me

# Voir les décisions contre votre IP
docker-compose exec crowdsec cscli decisions list | grep "<votre-ip>"

# Débloquer (demander à l'admin)
docker-compose exec crowdsec cscli decisions delete --ip <votre-ip>
```

### Puis-je avoir de faux positifs ?

**Oui**, surtout avec :
- Les scanners de vulnérabilité légitimes
- Les crawlers d'indexation
- Les outils de test de charge

**Solution** : Ajuster les seuils ou whitelister les IPs.

---
