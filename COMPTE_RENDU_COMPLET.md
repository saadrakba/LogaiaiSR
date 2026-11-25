# Compte rendu complet — Atelier 6 : transformer LogiAI en package

**Nom :** ................................

**Prénom :** ................................

**Groupe :** ................................

**Date :** 25 novembre 2026

---

## 1. Objectif de l'atelier

L'objectif était de transformer le projet LogiAI en véritable package Python installable. Les principales actions ont été :

- description du package dans `pyproject.toml`;
- installation en mode éditable;
- création de la commande terminal `logiai-predict`;
- construction d'un wheel;
- utilisation du package dans une image Docker;
- ajout d'une ADR et d'une étude de conception;
- vérification du fonctionnement avec les tests pytest.

Le projet utilisé se trouve dans :

```text
C:\Users\saadr\Desktop\logiai_Atelier4\logiai_Atelier4\starter2
```

---

## 2. Constater le problème d'import

Depuis la racine du projet :

```powershell
python -c "import logiai; print(logiai.__file__)"
```

Cette commande affiche le chemin du paquet lorsque le dossier du projet contient directement `logiai/`.

Depuis le dossier parent :

```powershell
cd ..
python -c "import logiai; print(logiai.__file__)"
```

L'import échoue avec :

```text
ModuleNotFoundError: No module named 'logiai'
```

### Conclusion

> Pour utiliser LogiAI, mon collègue doit installer le package avec pip ou ajouter le dossier du projet à son environnement Python.

---

## 3. Création du fichier `pyproject.toml`

Un fichier `pyproject.toml` a été créé à la racine du projet.

Il contient notamment :

- le nom du package : `logiai`;
- la version : `0.1.0`;
- la version minimale de Python : `>=3.10`;
- les dépendances principales : pandas, NumPy, scikit-learn, joblib et python-dotenv;
- les dépendances optionnelles `api` et `dev`;
- la découverte des modules dont le nom commence par `logiai`;
- la déclaration future ou effective de la commande `logiai-predict`.

Exemple de déclaration de la commande :

```toml
[project.scripts]
logiai-predict = "logiai.cli:main"
```

---

## 4. Test du package

Un fichier `tests/test_package.py` a été créé avec les tests suivants :

```python
from importlib.metadata import entry_points, version


def test_package_installe():
    assert version("logiai")


def test_commande_declaree():
    noms = {e.name for e in entry_points(group="console_scripts")}
    assert "logiai-predict" in noms


def test_cli_fichier_absent():
    from logiai.cli import main

    assert main(["fichier_inexistant.json"]) == 1
```

Avant l'installation du package, les tests de package échouent normalement. Après l'installation éditable et la déclaration de la commande, les tests passent.

---

## 5. Installation en mode éditable

L'environnement virtuel a été activé puis le package a été installé avec :

```powershell
.\.venv\Scripts\Activate.ps1
pip install -e ".[api,dev]"
```

Le résultat de la vérification est :

```text
Name: logiai
Version: 0.1.0
```

La commande suivante confirme également que le package est enregistré :

```powershell
pip show logiai
```

Après installation, l'import fonctionne depuis un dossier extérieur au projet.

### Mode éditable

Le mode éditable est utile pendant le développement :

- les modifications du code sont immédiatement visibles;
- il n'est pas nécessaire de reconstruire un wheel à chaque modification;
- Python trouve le package depuis son environnement virtuel.

---

## 6. Création de la commande `logiai-predict`

Le fichier `logiai/cli.py` a été créé.

Il :

1. reçoit le chemin d'un fichier JSON;
2. vérifie que le fichier existe;
3. lit le JSON;
4. utilise `Predictor` pour faire la prédiction;
5. calcule le pourcentage;
6. affiche `ALERTE` si la probabilité atteint le seuil;
7. retourne `0` en cas de succès et `1` en cas d'erreur.

La commande peut être utilisée avec :

```powershell
logiai-predict livraison.json
```

Résultat obtenu avec la livraison normale :

```text
Probabilité de retard : 73 % (ALERTE)
```

---

## 7. Test avec une pluie de 500 mm

Une copie de `livraison.json` a été créée avec une valeur de 500 mm pour `pluie_mm`.

### Résultat de la CLI

```text
Probabilité de retard : 100 % (ALERTE)
```

### Résultat de l'API

```text
422 Unprocessable Entity
```

### Explication

- La commande CLI appelle directement `Predictor.predict()`.
- Elle ne passe pas par le schéma Pydantic `Livraison`.
- L'API utilise `Livraison`, dont la borne maximale de `pluie_mm` est 100 mm.
- La valeur 500 mm est donc interdite par l'API.

Cette différence montre que la CLI ne remplace pas encore complètement la validation complète de l'API.

---

## 8. Construction du wheel

Les commandes utilisées sont :

```powershell
python -m pip install build
python -m build
python -m zipfile -l dist\logiai-0.1.0-py3-none-any.whl
```

La liste du wheel contient les fichiers Python du package `logiai`.

### Contenu inclus

Le wheel contient notamment :

- `logiai/__init__.py`;
- `logiai/cli.py`;
- `logiai/config.py`;
- `logiai/predict.py`;
- `logiai/train.py`;
- les modules de l'API.

### Contenu exclu

Le wheel ne contient pas :

- `models/`;
- `data/`;
- `tests/`;
- `.env`;
- les fichiers Docker;
- les fichiers de documentation.

---

## 9. Résolution de la racine externe

Dans `logiai/config.py`, la racine du projet est définie ainsi :

```python
import os
from pathlib import Path

ROOT = Path(os.environ.get("LOGIAI_HOME", Path.cwd()))
```

Cette modification permet :

- d'utiliser le dossier courant par défaut;
- de définir un autre emplacement avec `LOGIAI_HOME`;
- de retrouver les dossiers `data/` et `models/` hors du wheel.

---

## 10. Test du wheel dans un environnement vide

Dans un autre environnement, le wheel doit être installé avec :

```powershell
pip install C:\chemin\vers\dist\logiai-0.1.0-py3-none-any.whl
```

Premier lancement sans `LOGIAI_HOME` :

- le modèle n'est pas trouvé;
- l'erreur indique que le fichier du modèle est absent ou inaccessible;
- ce comportement est normal, car `models/` n'est pas inclus dans le wheel.

Deuxième lancement avec :

```powershell
set LOGIAI_HOME=C:\Users\saadr\Desktop\logiai_Atelier4\logiai_Atelier4\starter2
logiai-predict C:\Users\saadr\Desktop\logiai_Atelier4\logiai_Atelier4\starter2\livraison.json
```

Le lancement affiche alors une prédiction, car `LOGIAI_HOME` permet de retrouver le dossier `models/`.

---

## 11. Installation dans Docker

Le `Dockerfile` a été modifié pour remplacer la copie directe du dossier source par une installation du package.

```dockerfile
COPY pyproject.toml /tmp/build/
COPY logiai/ /tmp/build/logiai/
RUN pip install --no-cache-dir --no-deps /tmp/build \
    && rm -rf /tmp/build
```

Ensuite, les dossiers nécessaires sont encore copiés séparément :

```dockerfile
COPY data/ ./data/
COPY models/ ./models/
```

Cette organisation permet :

- d'obtenir un package LogiAI réel dans `site-packages`;
- de ne pas copier le dossier source `logiai/` dans `/app`;
- de conserver les données et modèles comme ressources externes.

Dans le conteneur, `logiai.__file__` doit être proche de :

```text
/usr/local/lib/python3.11/site-packages/logiai/__init__.py
```

Le dossier `/app` doit contenir `data/` et `models/`, mais pas `logiai/`.

---

## 12. Vérification Docker

La configuration Docker Compose a été validée avec :

```powershell
docker compose config
```

Les commandes suivantes doivent être utilisées après démarrage de Docker Desktop :

```powershell
docker compose up -d --build
docker compose ps
docker compose exec api python -c "import logiai; print(logiai.__file__)"
docker compose exec api ls /app
```

Résultat attendu :

- l'API est `healthy`;
- une clé absente donne `401`;
- une fausse clé donne `401`;
- une clé utilisateur valide donne `200`;
- la page sur le port `8080` fonctionne;
- `/app` contient `data/` et `models/`;
- `/app` ne contient pas `logiai/`.

Lors du test réalisé, Docker n'a pas démarré les conteneurs, car Docker Desktop n'était pas actif.

---

## 13. Résultats des tests Python

L'environnement a été installé avec :

```powershell
pip install -e ".[api,dev]"
```

Puis les tests ont été lancés avec :

```powershell
python -m pytest
```

Résultat final :

```text
54 tests réussis
54 passed
1 warning
```

Tests séparés possibles :

```powershell
python -m pytest tests/test_api.py
python -m pytest tests/test_casser.py
python -m pytest tests/test_limites.py
python -m pytest tests/test_package.py
python -m pytest tests/test_securite.py
```

Affichage détaillé :

```powershell
python -m pytest -v
```

---

## 14. ADR 005 — Packaging

Le fichier `docs/adr/005-packaging.md` contient les décisions suivantes.

### Pourquoi `>=` dans `pyproject.toml` ?

`>=` indique une compatibilité minimale. Il permet aux utilisateurs d'installer une version récente compatible du package.

Un choix avec une contrainte trop stricte ferait perdre la souplesse des mises à jour. En revanche, `requirements.txt` utilise les versions exactes pour rendre les environnements reproductibles.

### Pourquoi `==` dans `requirements.txt` ?

Les versions exactes permettent :

- de reproduire le même environnement;
- d'éviter les différences de résultats;
- d'utiliser les versions prévues par le Docker;
- de limiter les problèmes liés aux incompatibilités de dépendances.

### Le modèle fait-il partie du package ?

Non. Le modèle reste dans `models/`.

Le modèle reste séparé pour :

- éviter un wheel trop lourd;
- permettre son entraînement indépendant;
- remplacer le modèle sans reconstruire tout le package;
- conserver les données de fonctionnement hors du code Python.

---

## 15. Étude de conception

Le fichier `docs/etude-de-conception.md` contient les neuf titres demandés :

1. besoin;
2. données;
3. modèle et métriques;
4. critères d'acceptation;
5. cycle de vie;
6. architecture;
7. choix technologiques;
8. risques et éthique;
9. limites.

Aujourd'hui, seule la section architecture est renseignée.

L'architecture comprend :

- les données de livraison;
- le modèle `model.joblib`;
- le package Python `logiai`;
- la commande `logiai-predict`;
- l'API FastAPI;
- Nginx;
- la page web;
- les conteneurs Docker.

---

## 16. `.gitignore`

Le fichier `.gitignore` exclut notamment :

```text
.venv/
.env
data/*.csv
models/
dist/
build/
*.egg-info/
```

Ces éléments ne doivent pas être ajoutés au dépôt Git.

---

## 17. Différences techniques importantes

### Package Python et Docker

Un package Python contient les modules et les commandes Python.

Docker fournit tout un environnement d'exécution :

- le système;
- les dépendances;
- le package;
- les données;
- le modèle;
- l'API;
- le serveur web.

### Installation pour développer

```powershell
pip install -e .
```

Cette forme est adaptée au développement, car elle utilise directement les fichiers du projet.

### Installation pour livrer

Le wheel est adapté à la livraison :

```text
logiai-0.1.0-py3-none-any.whl
```

Il contient le code du package, mais pas nécessairement les modèles ou données externes.

---

## 18. Réponses au ticket de sortie

### 1. Docker et package : quelle différence ?

Un package Python distribue le code `logiai` et sa commande. Docker distribue tout l'environnement nécessaire pour exécuter l'application.

### 2. Quand utiliser `pip install -e .` ou wheel ?

- `pip install -e .` : développement;
- wheel : livraison et installation indépendante.

### 3. Qu'est-ce qui n'est pas dans le wheel ?

Le wheel ne contient pas :

- le modèle;
- les données;
- les tests;
- `.env`;
- les fichiers Docker.

Il faut livrer avec :

- `models/`;
- les données nécessaires;
- `LOGIAI_HOME`;
- les clés API dans les variables d'environnement.

### 4. Points restant flous

- démarrage complet de Docker Desktop;
- chemin exact de `logiai` dans le conteneur;
- vérification visuelle de la page sur le port 8080;
- test complet des clés API;
- validation du workflow Git et du pull request.

---

## 19. Résumé des livrables

Les éléments suivants ont été réalisés :

- `pyproject.toml`;
- `logiai/cli.py`;
- modification de `logiai/config.py`;
- `tests/test_package.py`;
- modification du `Dockerfile`;
- modification de `docker-compose.yml`;
- mise à jour du `.gitignore`;
- wheel construit dans `dist/`;
- `docs/adr/005-packaging.md`;
- `docs/etude-de-conception.md`;
- mise à jour du compte rendu;
- création du présent compte rendu complet.

### Résultat

Le packaging Python est opérationnel, la commande est créée et les tests passent. Le test Docker complet reste à exécuter avec Docker Desktop démarré.
