# Étude de conception

## besoin

## données

## modèle et métriques

## critères d'acceptation

## cycle de vie

## architecture

```text
[livraison JSON] -> [commande logiai-predict] -> [package logiai]
                                            -> [Predictor -> models/model.joblib]

[page web] -> [Nginx :8080] -> [API FastAPI :8000]
                                  |
                                  +-> [package logiai]
                                  +-> [Predictor -> models/model.joblib]

[data/livraisons.csv] -> [train.py] -> [models/model.joblib]

Conteneurs Docker:
- conteneur api: package logiai installé dans site-packages, data/ et models/ montés dans /app
- conteneur web: page statique servie derrière Nginx
- conteneur nginx: relais HTTP vers l'API
```

## choix technologiques

## risques et éthique

## limites
