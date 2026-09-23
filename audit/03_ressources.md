# Audit - ressources et sobriété

## Mesures du modèle historique

Les mesures ont été réalisées avec `psutil` et `perf_counter` dans le notebook. Elles décrivent l'environnement d'audit et non une production représentative.

| Mesure | Résultat |
|---|---:|
| Taille du modèle sur disque | **4,73 MB** |
| RSS avant inférence | **186,62 MB** |
| Temps pour 100 lignes | **5,1 ms** |
| Temps pour 1 000 lignes | **13,2 ms** |
| Temps pour 10 000 lignes | **70,1 ms** |
| Variation RSS après 10 000 lignes | environ **0,49 MB** |

Le coût d'inférence et l'empreinte du modèle sont modestes sur les volumes testés. Aucun test de concurrence, de pic de charge, de réseau ou de temps de chargement n'a été réalisé. Le temps d'entraînement du `train.py` historique n'a pas été mesuré séparément.

## Comparaison des modèles

| Modèle | ROC-AUC test | Entraînement | Inférence test | Taille sérialisée |
|---|---:|---:|---:|---:|
| Random Forest historique | 0,7195 | 362,1 ms | 13,9 ms | 4,37 MB |
| Régression logistique | **0,7364** | **6,9 ms** | **2,2 ms** | **0,0013 MB** |
| Gradient Boosting | 0,7199 | 1 004,9 ms | 15,7 ms | 0,34 MB |

La régression logistique est la plus légère et la plus rapide sur cette exécution, avec le meilleur ROC-AUC. La comparaison reste exploratoire, car une seule séparation train/test a été utilisée.

## Lecture de sobriété

Les chiffres appuient un constat de sobriété sur le stockage et le calcul. La régression logistique nécessite environ 3 600 fois moins de stockage et un temps d'entraînement environ 52 fois inférieur au Random Forest dans cette exécution.

Ces mesures ne donnent pas directement l'énergie, les émissions carbone ni le coût opérationnel complet, qui inclut notamment déploiement, supervision, maintenance, validation humaine et traçabilité.
