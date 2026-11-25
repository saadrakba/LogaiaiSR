# Compte rendu : tester, sécuriser et appeler l'API LogiAI

**Noms :** ............................  **Groupe :** ........

## Atelier 6 : packaging LogiAI

### Étape 1 - Import hors du dossier
- Dans le dossier du projet, `import logiai` affiche le chemin du paquet chargé.
- Depuis le dossier parent, l'import échoue avec `ModuleNotFoundError: No module named 'logiai'`.
- Phrase à retenir : « pour utiliser LogiAI, mon collègue doit installer le paquet avec pip ou ajouter le dossier du projet à son environnement Python ».

### Étape 4 - Pluie de 500 mm
- La commande CLI ne passe pas par le schéma Pydantic : avec la copie du JSON à 500 mm, elle affiche `100 % (ALERTE)`.
- La requête API reçoit `422 Unprocessable Entity`, car `pluie_mm` est limité à `100` mm par le schéma.
- La différence vient du point de validation : l'API applique les bornes métier avant la prédiction, tandis que la CLI appelle directement le modèle.

### Étape 5 - Wheel hors du projet
- Sans `LOGIAI_HOME`, l'environnement neuf échoue car le modèle n'est pas inclus dans le wheel : `Modèle non chargé ou fichier introuvable : /models/model.joblib` selon le chemin courant.
- Avec `LOGIAI_HOME` positionné sur le dossier du projet, la commande trouve `models/model.joblib` et affiche une prédiction.

### Étape 6 - Docker
- Dans le conteneur API, `logiai.__file__` doit pointer vers `site-packages`, par exemple `/usr/local/lib/python3.11/site-packages/logiai/__init__.py`.
- `/app` conserve `data/` et `models/`, mais ne contient plus le dossier source `logiai/`.

## 1. Casser son API (activité 1)
| Cas | Requête (corps ou URL) | Prévu | Obtenu |
|---|---|---|---|
| champ manquant | `/v1/predict`, poids absent | 422 | 422 |
| valeur absurde | `/v1/predict`, poids 99999 | 422 | 422 |
| texte au lieu d'un nombre | `/v1/predict`, distance `"loin"` | 422 | 422 |
| champ en trop | `/v1/predict`, champ `couleur` | 422 | 422 |
| mauvaise route | `/route-inconnue` | 404 | 404 |

- Une requête acceptée alors qu'elle devait être refusée ? Laquelle ?

## 2. Les 2 bugs (activité 2)
| Bug | Test qui l'a révélé | Comportement avant | Correction |
|---|---|---|---|
| 1 | `heure_depart=24` doit être refusé | La borne supérieure acceptait 24 | `le=23` |
| 2 | `nb_arrets=-1` doit être refusé | Aucun minimum n'était défini | `ge=0` |

- Sortie de `python -m pytest` à la fin : 51 tests réussis dans le virtualenv du projet.

## 3. La clé API (sprint 1)
- Ma fonction `verify_api_key()` (collez le code) :
```python
def verify_api_key(x_api_key: str | None = Header(default=None)) -> str:
    if not x_api_key:
        raise HTTPException(
            status_code=401,
            detail="Clé API absente",
            headers={"WWW-Authenticate": "ApiKey"},
        )
    if _egal(x_api_key, config.ADMIN_KEY):
        return "admin"
    if _egal(x_api_key, config.API_KEY):
        return "user"
    raise HTTPException(
        status_code=401,
        detail="Clé API invalide",
        headers={"WWW-Authenticate": "ApiKey"},
    )
```

| Requête | Code obtenu |
|---|---|
| `/v1/predict` sans clé | 401 |
| `/v1/predict` fausse clé | 401 |
| `/v1/predict` clé utilisateur | 200 |
| `/v1/model-info` clé utilisateur | 403 |
| `/v1/model-info` clé administrateur | 200 |
| `/health` sans clé | 200 |

- Preuve que `.env` n'est pas dans Git (sortie de `git status`) :

## 4. Le client web (sprint 2)
- Mon `fetch` (collez le code) :
```javascript
const reponse = await fetch(API_URL, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-API-Key": cle,
  },
  body: JSON.stringify(livraison),
});
const corps = await reponse.json();
return { status: reponse.status, corps };
```
- Ce qui s'affiche : livraison normale / pluie de 500 mm / fausse clé :
- Expérience CORS (page servie sur le port 5501) : que disent la page et la console (F12) ? Pourquoi `curl` marche-t-il quand même ?
- Où voyez-vous la clé dans F12 ?

## 5. Questions de compréhension
1. **401** signifie que l'authentification est absente ou invalide; **403** signifie que l'identité est connue mais n'a pas les droits; **422** signifie que les données sont mal formées ou hors bornes.

. Quelle est la différence entre les codes HTTP 401, 403 et 422 ?
- 401 Unauthorized : signifie que l'authentification est absente ou que les informations d'identification fournies sont invalides.
- 403 Forbidden : signifie que l'identité de l'utilisateur est connue, mais qu'il ne possède pas les autorisations nécessaires pour accéder à la ressource.
- 422 Unprocessable Entity : indique que les données envoyées sont mal formées ou ne respectent pas les contraintes de validation définies par l'API.


2. Elle doit rester dans `.env` pour éviter qu'un secret partagé ou historisé soit exposé dans le dépôt ou le code.

La clé API doit être conservée dans un fichier .env afin de protéger les informations sensibles. Cela évite d'exposer un secret partagé dans le code source ou dans un dépôt Git. Cette approche facilite également la gestion des configurations selon les environnements de développement et de production.


3. `compare_digest` compare en temps constant afin de limiter les attaques par mesure du temps; `==` peut révéler progressivement des caractères en arrêtant plus tôt la comparaison.

La fonction compare_digest effectue une comparaison en temps constant afin de limiter les attaques par analyse temporelle (Timing Attacks). Contrairement à l'opérateur ==, qui peut arrêter la comparaison dès qu'une différence est détectée, cette fonction réduit les risques de divulgation progressive d'informations sur la clé secrète.


4. `curl` n'est pas soumis à la politique CORS d'un navigateur; CORS protège les pages web lancées dans un navigateur, pas les clients HTTP hors navigateur.

curl est un client HTTP qui n'est pas soumis aux restrictions CORS appliquées par les navigateurs. CORS (Cross-Origin Resource Sharing) est un mécanisme de sécurité des navigateurs qui contrôle les communications entre différentes origines. Ainsi, une requête peut fonctionner avec curl tout en étant bloquée par le navigateur si la configuration CORS de l'API n'autorise pas l'origine concernée.


5. Le navigateur améliore l'expérience utilisateur mais reste modifiable; seule l'API doit être autoritaire et doit valider toutes les entrées qu'elle reçoit.

La validation côté navigateur améliore l'expérience utilisateur en détectant rapidement les erreurs de saisie. Cependant, les contrôles côté client peuvent être contournés ou modifiés. L'API doit donc rester la source d'autorité et effectuer systématiquement la validation de toutes les données reçues afin de garantir la sécurité, la cohérence et la fiabilité du système.

## 6. Lien de la pull request


!