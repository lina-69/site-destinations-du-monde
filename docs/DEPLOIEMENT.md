# 🐍 **Guide de déploiement Wagtail sur PythonAnywhere**

## ⚠️ Informations à personnaliser

Remplace dans toutes les commandes :

* **TON_COMPTE** → ton nom de compte GitHub
* **TON_REPO** → nom du dépôt GitHub
* **TON_PROJET** → nom du projet Django/Wagtail (dossier contenant `settings/`)
* **USERNAME** → nom de ton compte PythonAnywhere

# 0. Préparer le projet pour le déploiement

### 1) Code sur GitHub

Votre projet Wagtail doit être sur GitHub, avec un `.gitignore` qui exclut les fichiers locaux :

```
venv/
__pycache__/
db.sqlite3
media/
```

### 2) Préparer requirements.txt

Voici un exemple adapté au déploiement :

```
Django>=5.2,<5.3
wagtail>=7.2,<7.3

python-dotenv

gunicorn
whitenoise

psycopg[binary]
Pillow
```

# 1. Créer un compte PythonAnywhere

1. Aller sur :
   [https://www.pythonanywhere.com/registration/register/beginner/](https://www.pythonanywhere.com/registration/register/beginner/)
2. Choisir votre **USERNAME** (exemple : `monapp2025`)
   ➜ Votre site sera accessible ici :
   `https://USERNAME.pythonanywhere.com/`
3. Valider l’email.


# 2. Créer une nouvelle Web App

1. Aller dans **Web** (menu en haut).
2. Cliquer sur **Add a new web app**.
3. Accepter le domaine proposé :
   `USERNAME.pythonanywhere.com`
4. Choisir :

   * *Manual configuration*
   * *Python 3.13* (ou version récente)
5. Valider.

# 3. Cloner votre projet GitHub

1. Aller dans **Files**
2. Cliquer sur **Open Bash console here**
3. Taper :

```bash
git clone https://github.com/TON_COMPTE/TON_REPO.git
```

Cela crée le dossier :

```
/home/USERNAME/TON_REPO/
```

# 4. Créer un virtualenv + installer les dépendances

Dans la console Bash :

```bash
cd /home/USERNAME
python3 -m venv venv
source venv/bin/activate
pip install -r TON_REPO/requirements.txt
```

# 5. Configurer les settings de production
## 5.1 Configuration générale

Dans **Files → TON_REPO → TON_PROJET → settings**, vous devez avoir :

* `base.py`
* `dev.py`
* `production.py`

### a) Copier la SECRET_KEY

Dans `dev.py` ou `base.py` :

```python
SECRET_KEY = "votre_cle_secrete"
```

Copiez-la.

### b) Modifier `production.py`

Exemple recommandé :

```python
from .base import *

DEBUG = False

# Optionnel : désactiver ManifestStaticFilesStorage pour simplifier la formation
# STORAGES["staticfiles"]["BACKEND"] = "django.contrib.staticfiles.storage.ManifestStaticFilesStorage"

SECRET_KEY = "COLLER_ICI_LA_SECRET_KEY"

ALLOWED_HOSTS = [
    "USERNAME.pythonanywhere.com",
    "localhost",
    "127.0.0.1",
]

try:
    from .local import *
except ImportError:
    pass
```

Remplacer **USERNAME** par votre nom PythonAnywhere.

## **5.2 Configurer l’envoi d’emails en production (Gmail)**

Si votre page **Contact** envoie un email en local, vous devez reproduire la configuration en **production** sur PythonAnywhere.

Pour cela, deux étapes sont nécessaires et complémentaires :
➡️ **Étape 1 : configurer `production.py`**

➡️ **Étape 2 : définir les variables d’environnement sur PythonAnywhere**


### **Étape 1 — Ajouter la configuration email dans `production.py`**

Dans `TON_PROJET/settings/production.py`, ajoutez en haut :

```python
import os
```

Puis ajoutez la configuration email :

```python
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "smtp.gmail.com"
EMAIL_PORT = 587
EMAIL_USE_TLS = True

EMAIL_HOST_USER = os.environ.get("EMAIL_HOST_USER")
EMAIL_HOST_PASSWORD = os.environ.get("EMAIL_HOST_PASSWORD")
```

Cette configuration indique à Django d'utiliser Gmail et de récupérer les identifiants depuis les variables d’environnement du serveur.

### **Étape 2 — Définir les variables d’environnement dans PythonAnywhere**

1. Aller dans **Web → Environment variables**
2. Ajouter les deux variables suivantes :

```
EMAIL_HOST_USER=ton_email@gmail.com
EMAIL_HOST_PASSWORD=TON_MDP_APPLICATION_GMAIL
```

➡️ Remplacez par votre adresse Gmail et *votre mot de passe d’application Gmail*.

➡️ Ne jamais mettre d’espaces avant ou après le `=`.

3. Enregistrer puis cliquer sur **Reload** de la Web App.

### ⚠️ **Points importants**

#### 1️⃣ Utilisez obligatoirement un **mot de passe d’application Gmail**

Ce n’est pas votre mot de passe habituel.
Il faut le créer dans :
**Gérer votre compte Google → Sécurité → Mots de passe d’application**

#### 2️⃣ Ne jamais écrire un mot de passe directement dans le code

Toujours utiliser :

```python
os.environ.get("EMAIL_HOST_PASSWORD")
```

#### 3️⃣ Vérifier que la configuration fonctionne

Dans une console Bash sur PythonAnywhere :

```bash
python manage.py shell
```

Puis :

```python
from django.conf import settings
print(settings.EMAIL_HOST_USER)
print(settings.EMAIL_HOST_PASSWORD is not None)
```

Vous devez obtenir :

```
ton_email@gmail.com
True
```

Votre formulaire **Contact** peut maintenant envoyer des emails en production, exactement comme en local.

# 6. Créer la base de données de production

Dans une console Bash :

```bash
cd /home/USERNAME/TON_REPO/
source ../venv/bin/activate
python manage.py migrate
```

# 7. Créer le superuser

```bash
python manage.py createsuperuser
```

⚠️ Si Django affiche :

> Bypass password validation and create user anyway? (y/N)

Répondez `y`.


# 8. Collecter les fichiers statiques

```bash
python manage.py collectstatic
```

Taper `yes` si demandé.


# 9️. Configurer la Web App (onglet Web)

## A) Associer le virtualenv

Dans **Web → Virtualenv**, mettre :

```
/home/USERNAME/venv
```

Valider (bouton ✔).


## B) Configurer le fichier WSGI

Dans **Web**, section **WSGI configuration file** → ouvrir le fichier.

Remplacer tout par :

```python
import os
import sys

project_path = "/home/USERNAME/TON_REPO"
if project_path not in sys.path:
    sys.path.append(project_path)

os.environ.setdefault(
    "DJANGO_SETTINGS_MODULE",
    "TON_PROJET.settings.production"
)

from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```

➡ Remplacer :

* **USERNAME**
* **TON_REPO**
* **TON_PROJET.settings.production**

Sauvegarder.


# 10. Configurer les Static files et Media files

### Vérifier dans `base.py` :

```python
STATIC_URL = "/static/"
STATIC_ROOT = os.path.join(BASE_DIR, "static")

MEDIA_URL = "/media/"
MEDIA_ROOT = os.path.join(BASE_DIR, "media")
```

### Ensuite dans **Web → Static files** :

Ajouter deux lignes :

1) **Fichiers statiques**

* URL : `/static/`
* Directory : `/home/USERNAME/TON_REPO/static/`

2) **Fichiers médias (images uploadées)**

* URL : `/media/`
* Directory : `/home/USERNAME/TON_REPO/media/`

Enregistrer avec **Save**.


# 11. Redémarrer la Web App

En haut → bouton **Reload**.


# 12. Tester l’application

👉 Site public :
`https://USERNAME.pythonanywhere.com/`

👉 Interface Admin :
`https://USERNAME.pythonanywhere.com/admin/`

Connectez-vous avec votre superuser.

# 13. Finaliser le contenu en production

⚠️ IMPORTANT
Les contenus créés en local ne sont **pas copiés** automatiquement.
La base SQLite de production est **neuve**.

Créer les pages en production via `/admin/` :

1. Menu **Pages**
2. Cliquer sur **Home**
3. **Add child page**
4. Choisir un type (Blog, About, Contact…)
5. Remplir → **Publish**

Pour les images :

* **Images → Add image**
* Puis insérer l’image dans vos pages.


# 🎉 Votre site Wagtail est maintenant en production !

