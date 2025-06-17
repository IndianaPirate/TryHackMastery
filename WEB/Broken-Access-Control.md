## 🟢 **Niveau 1 : Facile – Accès horizontal non contrôlé (IDOR simple)**

### 🔸 Théorie :

**IDOR (Insecure Direct Object Reference)** permet d'accéder à une ressource sans contrôle d’autorisation, simplement en changeant un identifiant dans l’URL ou le corps de la requête.

### 🔸 Exemple :

```
GET /api/users/1234/profile
```

Si l’utilisateur 5678 appelle cette URL et récupère les infos de 1234, c’est une faille.

### 🔸 Simulation :

Dans Flask :

```python
@app.route("/users/<int:user_id>")
def get_user(user_id):
    user = db.get_user_by_id(user_id)
    return jsonify(user.to_dict())
```

### 🔸 Comment tester :

* Crée 2 comptes.
* Avec le second, modifie l’URL pour accéder à l’ID du premier.

### 🔸 Comment corriger :

```python
if current_user.id != user_id:
    return abort(403)
```

---

## 🟡 **Niveau 2 : Moyen – Accès vertical non contrôlé (élévation de privilèges)**

### 🔸 Théorie :

Un utilisateur standard parvient à exécuter des actions réservées à des admins, faute de vérification des rôles.

### 🔸 Exemple :

```http
POST /api/users/ban?id=1234
```

L’utilisateur `john` envoie cette requête et bannit quelqu’un sans être admin.

### 🔸 Code vulnérable :

```python
@app.route("/api/users/ban")
def ban_user():
    user_id = request.args.get("id")
    user = db.get_user_by_id(user_id)
    user.banned = True
    db.save(user)
```

### 🔸 Correction :

```python
if not current_user.is_admin:
    return abort(403)
```

---

## 🟠 **Niveau 3 : Avancé – Contrôle par champ non protégé (Mass Assignment)**

### 🔸 Théorie :

L’utilisateur soumet des champs normalement réservés à l’admin (ex : `is_admin: true`) dans un formulaire ou JSON, et ils sont acceptés sans vérification.

### 🔸 Exemple :

```json
PATCH /api/users/1234
{
  "email": "attacker@evil.com",
  "is_admin": true
}
```

### 🔸 Code vulnérable :

```python
def update_user(id):
    user = db.get_user_by_id(id)
    for key, value in request.json.items():
        setattr(user, key, value)
    db.save(user)
```

### 🔸 Correction :

* Filtrer les champs autorisés :

```python
allowed_fields = ['email', 'avatar']
for key in allowed_fields:
    if key in request.json:
        setattr(user, key, request.json[key])
```

---

## 🔴 **Niveau 4 : Très avancé – Contrôle d’accès indirect (via actions sur d’autres objets)**

### 🔸 Théorie :

L'utilisateur agit sur une ressource "liée", sans vérifier s’il en est bien propriétaire. Exemple : commenter un ticket ou modifier un fichier joint à un projet.

### 🔸 Exemple :

```http
POST /api/comments
{
  "ticket_id": 10,
  "text": "Exploit!"
}
```

L’utilisateur n’a **pas accès au ticket 10**, mais peut commenter dessus car aucune vérification n’est faite sur l’accès au `ticket_id`.

### 🔸 En pratique :

* Modèle `Comment` associé à `Ticket` (foreign key).
* Attaquant crée un `Comment` sans que l’API vérifie s’il a droit d’accéder à ce ticket.

### 🔸 Correction :

Avant tout traitement :

```python
ticket = db.get_ticket(ticket_id)
if ticket.owner_id != current_user.id:
    abort(403)
```

---

## 🟣 **Niveau 5 : Difficile à détecter – Contrôle d’accès dans des microservices / GraphQL / cache**

### 🔸 Théorie :

Dans une architecture complexe (GraphQL, API Gateway, microservices), un service interne omet les contrôles d’accès en supposant que l’authentification est déjà faite.

### 🔸 Exemples :

* **GraphQL** : Attaquant écrit une requête `nested` ciblant des objets sensibles non filtrés.
* **Cache partagé** : Le cache retourne une donnée d’un autre utilisateur.
* **Service d’export** : Attaquant force l’export de données en devinant des IDs (ex: export PDF d’un ticket support qui ne lui appartient pas).

### 🔸 Simulation :

Dans un service GraphQL mal conçu :

```graphql
query {
  user(id: 1) {
    iban
    email
  }
}
```

Si aucune vérification d’ownership, un utilisateur peut cibler n’importe quel `user`.

---

## 🛡️ Bonnes pratiques générales (tous niveaux)

| Niveau | Pratique de sécurité                                        | Outils associés                                |
| ------ | ----------------------------------------------------------- | ---------------------------------------------- |
| 🟢     | Vérifier ownership (`current_user.id == resource.owner_id`) | Flask / Django middleware                      |
| 🟡     | Vérifier les rôles (`is_admin`, `permissions`)              | Casbin, Oso, Keycloak                          |
| 🟠     | Éviter la mass assignment                                   | Pydantic, serializers stricts                  |
| 🔴     | Appliquer les contrôles à tous les liens indirects          | Audit de code, tests fonctionnels              |
| 🟣     | Sécuriser les communications inter-services                 | JWT, mutual TLS, API Gateway, Zero Trust model |

---

## 🎓 Entraînement pratique proposé

| Niveau | Environnement                                                                       |
| ------ | ----------------------------------------------------------------------------------- |
| 🟢🟡   | Crée une mini-API Flask avec 2 users et routes `GET /profile`, `POST /admin/action` |
| 🟠     | Ajoute une route `PATCH /me` avec un champ interdit                                 |
| 🔴     | Crée des objets liés (commentaires → posts)                                         |
| 🟣     | Déploie 2 microservices Docker et simule un appel sans vérif d’accès                |
