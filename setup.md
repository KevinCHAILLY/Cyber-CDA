# Guide de Configuration - Lab WAF ModSecurity + DVWA

Ce guide explique comment installer et configurer le laboratoire de cybersécurité avec OWASP ModSecurity CRS et DVWA.

---

## Table des matières

1. [Prérequis](#prérequis)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [Démarrage](#démarrage)
5. [Dépannage](#dépannage)

---

## Prérequis

### Logiciels requis

- **Docker** (version 20.10+ recommandée)
- **Docker Compose** (version 1.29+ ou 2.x)

### Vérification de l'installation

```bash
# Vérifier Docker
docker --version

# Vérifier Docker Compose
docker-compose --version

# Vérifier que Docker fonctionne
docker run hello-world
```

---

## Installation

### 1. Cloner ou télécharger les fichiers

Assurez-vous d'avoir les fichiers suivants dans votre répertoire :

```
.
├── docker-compose.yml          # Version avancée avec logs détaillés
├── docker-compose.simple.yml   # Version simplifiée
├── modsecurity/
│   ├── custom-rules.conf       # Règles personnalisées
│   └── exclusions.conf         # Exclusions pour DVWA
└── setup.md                    # Ce fichier
```

### 2. Structure des services

#### Version simple ([`docker-compose.simple.yml`](docker-compose.simple.yml))

Idéale pour démarrer rapidement :

- ModSecurity CRS avec configuration par défaut
- DVWA (Damn Vulnerable Web Application)
- MySQL pour la base de données

#### Version avancée ([`docker-compose.yml`](docker-compose.yml))

Inclut des fonctionnalités supplémentaires :

- Logs détaillés en format JSON
- Montage de règles personnalisées
- Volumes pour la persistance des logs
- Configuration fine des seuils de détection

---

## Configuration

### Configuration DVWA

Par défaut, DVWA utilise ces identifiants :

| Paramètre | Valeur |
|-----------|--------|
| Database Server | `db` |
| Database Name | `dvwa` |
| Database User | `dvwa` |
| Database Password | `p@ssw0rd` |

### Configuration ModSecurity CRS

#### Variables d'environnement principales

| Variable | Description | Valeur par défaut |
|----------|-------------|-------------------|
| `PARANOIA` | Niveau de sensibilité | `1` |
| `ANOMALY_INBOUND` | Seuil d'anomalie entrante | `5` |
| `ANOMALY_OUTBOUND` | Seuil d'anomalie sortante | `4` |
| `BACKEND` | URL du backend | `http://dvwa:80` |
| `MODSEC_RULE_ENGINE` | Moteur de règles | `On` |

#### Niveaux de Paranoia

```yaml
# Dans docker-compose.yml
environment:
  - PARANOIA=1  # Basique - recommandé pour débuter
  # - PARANOIA=2  # Renforcé
  # - PARANOIA=3  # Strict
  # - PARANOIA=4  # Maximum
```

---

## Démarrage

### Démarrage rapide (version simple)

```bash
# Démarrer tous les services
docker-compose -f docker-compose.simple.yml up -d

# Vérifier l'état
docker-compose -f docker-compose.simple.yml ps

# Voir les logs
docker-compose -f docker-compose.simple.yml logs -f
```

### Démarrage avec logs avancés

```bash
# Démarrer la version complète
docker-compose up -d

# Voir les logs ModSecurity en temps réel
docker-compose logs -f modsecurity-crs
```

### Configuration initiale de DVWA

1. **Accéder à l'application** : `http://localhost`
2. **Créer la base de données** : Cliquez sur "Create / Reset Database"
3. **Se connecter** :
   - Username : `admin`
   - Password : `password`
4. **Configurer la sécurité** : Allez dans "DVWA Security" → Réglez sur "Low"

---

## Commandes utiles

### Gestion des containers

```bash
# Arrêter les services
docker-compose down

# Arrêter et supprimer les volumes (réinitialisation complète)
docker-compose down -v

# Redémarrer un service spécifique
docker-compose restart modsecurity-crs

# Voir les logs d'un service
docker-compose logs modsecurity-crs

# Entrer dans un container
docker exec -it modsecurity_waf /bin/bash
docker exec -it dvwa_app /bin/bash
```

### Accès aux logs

```bash
# Logs ModSecurity (version avancée)
docker exec -it modsecurity_waf tail -f /var/log/modsecurity/audit.log

# Logs Apache
docker exec -it modsecurity_waf tail -f /var/log/apache2/error.log
```

### Surveillance

```bash
# Utilisation des ressources
docker stats

# Processus dans un container
docker top modsecurity_waf
```

---

## Dépannage

### Problème : Les containers ne démarrent pas

**Symptôme** : `docker-compose up` échoue ou les containers s'arrêtent immédiatement.

**Solutions** :

```bash
# Vérifier les logs d'erreur
docker-compose logs

# Vérifier les ports utilisés
netstat -tlnp | grep :80
# ou
lsof -i :80

# Arrêter les services existants sur le port 80
sudo systemctl stop apache2
sudo systemctl stop nginx
```

### Problème : DVWA ne se connecte pas à la base de données

**Symptôme** : Erreur "Could not connect to the database".

**Solutions** :

```bash
# Vérifier que le container MySQL est démarré
docker-compose ps

# Voir les logs MySQL
docker-compose logs db

# Redémarrer les services dans l'ordre
docker-compose down
docker-compose up -d db
sleep 10
docker-compose up -d dvwa modsecurity-crs
```

### Problème : Le WAF bloque tout accès à DVWA

**Symptôme** : Erreurs 403 sur toutes les pages.

**Solutions** :

```bash
# Vérifier les règles activées
docker exec -it modsecurity_waf cat /etc/modsecurity.d/owasp-crs/crs-setup.conf | grep -i paranoia

# Modifier le niveau de paranoïa
# Éditer docker-compose.yml et changer PARANOIA=1
# Puis redémarrer
docker-compose down
docker-compose up -d
```

### Problème : Les logs ne sont pas accessibles

**Symptôme** : Impossible de voir les logs ModSecurity.

**Solutions** :

```bash
# Vérifier que le volume est monté (version avancée)
docker inspect modsecurity_waf | grep -A 5 "Mounts"

# Accéder aux logs directement
docker exec -it modsecurity_waf ls -la /var/log/modsecurity/

# Copier les logs localement
docker cp modsecurity_waf:/var/log/modsecurity/audit.log ./audit.log
```

### Problème : DVWA affiche des erreurs PHP

**Solutions** :

```bash
# Vérifier la configuration PHP
docker exec -it dvwa_app cat /var/www/html/config/config.inc.php

# Redémarrer DVWA
docker-compose restart dvwa
```

---

## Configuration réseau

### Ports utilisés

| Service | Port externe | Port interne | Description |
|---------|--------------|--------------|-------------|
| ModSecurity | 80 | 80 | Accès web via WAF |
| MySQL | - | 3306 | Base de données (interne) |
| DVWA | - | 80 | Application (interne) |

### Accès direct à DVWA (sans WAF)

Pour tester DVWA sans passer par le WAF :

```bash
# Modifier docker-compose.simple.yml pour exposer DVWA
# Ajouter sous le service dvwa :
ports:
  - "8080:80"

# Puis accéder à http://localhost:8080
```

---

## Sauvegarde et restauration

### Sauvegarder la base de données

```bash
# Créer un dump
docker exec dvwa_db mysqldump -u root -prootpass dvwa > dvwa_backup.sql
```

### Restaurer la base de données

```bash
# Restaurer depuis un dump
docker exec -i dvwa_db mysql -u root -prootpass dvwa < dvwa_backup.sql
```

---

## Sécurité

### ⚠️ Avertissements

- Ce lab est destiné à un usage **éducatif uniquement**
- **Ne déployez pas** cette configuration en production
- Les mots de passe sont faibles par design (pour le TP)
- DVWA contient des vulnérabilités intentionnelles

### Bonnes pratiques

1. **Isoler le réseau** : Le lab utilise un réseau Docker isolé
2. **Ne pas exposer** les ports internes (MySQL, DVWA direct)
3. **Surveiller les logs** pour détecter les activités suspectes
4. **Nettoyer** après utilisation avec `docker-compose down -v`

---

## Ressources supplémentaires

- [Documentation Docker](https://docs.docker.com/)
- [Documentation ModSecurity](https://github.com/SpiderLabs/ModSecurity/wiki)
- [OWASP CRS Documentation](https://coreruleset.org/docs/)
- [DVWA GitHub](https://github.com/digininja/DVWA)

---

## Support

En cas de problème persistant :

1. Vérifiez les logs : `docker-compose logs`
2. Consultez la section [Dépannage](#dépannage)
3. Vérifiez les issues sur les dépôts officiels des projets utilisés