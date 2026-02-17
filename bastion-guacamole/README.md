# Bastion Guacamole - Accès distant sécurisé

Ce projet déploie un **bastion d'accès** basé sur [Apache Guacamole](https://guacamole.apache.org/), une solution de passerelle d'accès distant sans client (clientless).

---

## 🎯 Objectif pédagogique

Ce lab permet aux étudiants de :
- Comprendre le concept de **bastion d'accès**
- Découvrir les solutions de **passerelle d'accès distant**
- Apprendre à configurer un accès sécurisé aux ressources internes
- Maîtriser Docker Compose avec les bonnes pratiques

---

## 🏗️ Architecture

```
┌─────────────────┐
│   Utilisateur   │
└────────┬────────┘
         │ HTTPS (navigateur)
         ▼
┌─────────────────────┐
│  Guacamole Web App  │  Port 8080
│  (Interface HTML5)  │
└────────┬────────────┘
         │
         ├──────────────────┐
         │                  │
         ▼                  ▼
┌──────────────┐    ┌──────────────┐
│    guacd     │    │  PostgreSQL  │
│  (Démon)     │    │  (Base de    │
│              │    │   données)   │
└──────────────┘    └──────────────┘
```

### Composants

| Service | Rôle | Image |
|---------|------|-------|
| **guacamole** | Interface web HTML5 | `guacamole/guacamole:1.5.3` |
| **guacd** | Démon de connexion | `guacamole/guacd:1.5.3` |
| **guacamole-db** | Base de données | `postgres:15-alpine` |

---

## 🚀 Démarrage rapide

### Prérequis

- Docker Engine 20.10+
- Docker Compose 1.29+ ou 2.x

### Lancement

```bash
# Se placer dans le répertoire
cd bastion-guacamole

# Démarrer les services
docker-compose up -d

# Vérifier que tout est démarré
docker-compose ps
```

### Accès

- **URL** : `http://localhost:8080/guacamole/`
- **Identifiants par défaut** :
  - Username : `guacadmin`
  - Password : `guacadmin`

> ⚠️ **Important** : Changez le mot de passe par défaut immédiatement !

---

## 📋 Configuration

### Variables d'environnement

| Variable | Description | Valeur par défaut |
|----------|-------------|-------------------|
| `GUACD_HOSTNAME` | Hôte du démon guacd | `guacd` |
| `POSTGRES_HOSTNAME` | Hôte de la base de données | `guacamole-db` |
| `POSTGRES_DATABASE` | Nom de la base | `guacamole_db` |
| `POSTGRES_USER` | Utilisateur PostgreSQL | `guacamole_user` |
| `POSTGRES_PASSWORD` | Mot de passe PostgreSQL | `guacamole_password` |

### Bonnes pratiques appliquées

✅ **Healthchecks** : Chaque service a un healthcheck pour vérifier son état  
✅ **Dépendances** : Les services démarrent dans le bon ordre (db → guacd → guacamole)  
✅ **Réseau isolé** : Un réseau Docker dédié (`guacamole_network`)  
✅ **Volumes nommés** : Persistance des données avec noms explicites  
✅ **Restart policy** : `unless-stopped` pour une haute disponibilité  
✅ **Versions fixes** : Images avec tags de version (pas `latest`)  

---

## 🛠️ Commandes utiles

```bash
# Démarrer
docker-compose up -d

# Arrêter
docker-compose down

# Voir les logs
docker-compose logs -f

# Logs d'un service spécifique
docker-compose logs -f guacamole

# Redémarrer un service
docker-compose restart guacamole

# Accès shell à la base de données
docker exec -it guacamole_db psql -U guacamole_user -d guacamole_db
```

---

## 📚 Utilisation pédagogique

### Exercice 1 : Configuration initiale

1. Connectez-vous avec `guacadmin` / `guacadmin`
2. Changez le mot de passe par défaut
3. Explorez l'interface d'administration

### Exercice 2 : Créer une connexion SSH

1. Allez dans **Settings** → **Connections** → **New Connection**
2. Configurez :
   - **Name** : `Serveur Test`
   - **Protocol** : `SSH`
   - **Hostname** : `[IP_DU_SERVEUR]`
   - **Port** : `22`
   - **Username** : `[VOTRE_USER]`
3. Testez la connexion

### Exercice 3 : Créer une connexion RDP

1. Créez une nouvelle connexion
2. Sélectionnez le protocole **RDP**
3. Configurez les paramètres d'une machine Windows
4. Testez l'accès bureau à distance

### Exercice 4 : Gestion des utilisateurs

1. Créez un nouvel utilisateur
2. Attribuez-lui des permissions sur une connexion
3. Testez la connexion avec ce nouvel utilisateur

---

## 🔒 Sécurité

### ⚠️ Avertissements

- Ce lab est destiné à un **usage éducatif uniquement**
- Les mots de passe par défaut doivent être changés
- N'exposez pas ce service sur Internet sans protection (WAF, VPN, etc.)

### Recommandations pour la production

- Utiliser **HTTPS** avec un reverse proxy (Nginx/Traefik)
- Configurer l'**authentification LDAP/Active Directory**
- Activer **2FA** (TOTP)
- Restreindre l'accès par **IP** ou **VPN**
- Changer les mots de passe par défaut
- Utiliser des **secrets Docker** pour les credentials

---

## 🐛 Dépannage

### Problème : La base de données ne démarre pas

```bash
# Vérifier les logs
docker-compose logs guacamole-db

# Réinitialiser
docker-compose down -v
docker-compose up -d
```

### Problème : Guacamole ne se connecte pas à la base

Attendez que la base soit prête (healthcheck) avant de démarrer guacamole.

### Problème : Impossible de se connecter à une machine distante

- Vérifiez que la machine cible est accessible
- Vérifiez les credentials
- Vérifiez les logs : `docker-compose logs guacd`

---

## 📖 Ressources

- [Documentation Guacamole](https://guacamole.apache.org/doc/)
- [Docker Hub Guacamole](https://hub.docker.com/r/guacamole/guacamole)
- [PostgreSQL Docker](https://hub.docker.com/_/postgres)

---

## 🧹 Nettoyage

```bash
# Arrêter et supprimer les volumes
docker-compose down -v

# Supprimer les images
docker rmi guacamole/guacamole:1.5.3 guacamole/guacd:1.5.3 postgres:15-alpine
```

---

**Bon apprentissage ! 🎓**