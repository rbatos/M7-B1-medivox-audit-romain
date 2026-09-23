# Audit - technique

## Architecture et couplage

L'architecture repose sur `train.py`, `predict.py` et un modèle `joblib`. La chaîne est compréhensible, mais peu modulaire : la préparation des features est dupliquée entre l'entraînement et l'inférence, sans pipeline partagé ni schéma versionné. L'ordre des colonnes et l'encodage manuel de `sexe` créent un couplage implicite entre les scripts et le modèle.

Le modèle est chargé depuis un chemin codé en dur dans `predict.py`, dépendant du répertoire courant. Un déplacement du fichier ou un changement de machine peut interrompre l'inférence. Les métadonnées du modèle, du dataset, du seuil et des dépendances ne sont pas documentées dans le fichier observé.

## Sécurité et validation

- `train.py` contient un mot de passe de production en clair.
- `predict.py` convertit directement les arguments en `int` et `float`.
- aucune validation de schéma, de bornes, de valeurs manquantes ou de cohérence n'est présente.
- aucune gestion d'erreur explicite n'est prévue.
- les colonnes texte sont supprimées sans pipeline d'encodage documenté.
- `sexe` est encodé manuellement en 0/1 sans justification visible.

Une entrée invalide peut donc provoquer une panne ou une prédiction non fiable.

## Déploiement et traçabilité

Les commentaires du code indiquent un appel en production via SSH et un déploiement manuel par `scp`. Le dépôt ne montre pas de versionnement du modèle, de contrôle d'intégrité, de rotation des secrets, de retour arrière ou de journalisation des entrées et sorties.

## Scalabilité et points de rupture

Les mesures du notebook donnent **5,1 ms** pour 100 lignes, **13,2 ms** pour 1 000 et **70,1 ms** pour 10 000. Le modèle pèse **4,73 MB**. Ces résultats sont favorables sur les volumes testés, mais ne couvrent pas la concurrence, les pics de charge, le réseau ou le chargement du modèle.

Le fichier modèle local et la machine qui l'héberge constituent un **SPOF** (*Single Point of Failure*, point de défaillance unique). Leur indisponibilité bloque les prédictions. L'absence de validation, de supervision et de logs retarde également la détection d'un incident.

## Conclusion

Le principal risque technique ne vient pas du coût de calcul, mais de la fragilité du déploiement, des secrets exposés, de l'absence de validation et de la faible traçabilité.
