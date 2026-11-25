# ADR 005 : conditionnement du package LogiAI

La dépendance déclarée avec `>=` dans `pyproject.toml` laisse setuptools choisir une version compatible récente sans bloquer les mises à jour de correctifs. Un `==` imposé partout perdrait la souplesse de consommation du package, tandis que `requirements.txt` fixe des versions exactes pour rendre les environnements Docker et les preuves reproductibles.

Le modèle reste à côté du package dans `models/`, livré avec les données de fonctionnement via `LOGIAI_HOME` et les volumes ou fichiers de déploiement. L'inclure dans le wheel ferait gonfler le paquet, figerait des artefacts sans traçabilité et compliquerait leur mise à jour indépendante.

Le code Python est donc distribué comme `logiai`, tandis que les artefacts runtime restent des ressources externes du déploiement.
