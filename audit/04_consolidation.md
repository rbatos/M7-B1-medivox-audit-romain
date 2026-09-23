# Audit - consolidation des risques

Les indicateurs sont classés du plus critique au moins critique. Le niveau 🔴 signale un risque pouvant affecter directement une décision médicale, la confidentialité ou la disponibilité du service. 🟠 signale une fragilité importante. 🟡 signale un point à confirmer ou un risque limité.

| Indicateur | Sévérité | Conséquence client |
|---|---|---|
| Secret de production en clair dans `train.py` | 🔴 | compromission potentielle de la production |
| Décision médicale sans supervision, log ni traçabilité | 🔴 | décision impossible à reconstituer ou contester |
| DI sexe **0,291**, FNR femmes **0,680**, FPR hommes **0,345** | 🔴 | effet défavorable dépendant du bénéfice ou de la contrainte associée à la décision |
| Absence de validation des entrées et de gestion d'erreur | 🔴 | panne ou prédiction non fiable sur entrée incohérente |
| Modèle local et machine unique comme SPOF | 🔴 | indisponibilité pouvant bloquer toutes les prédictions |
| Accuracy historique mesurée sur l'entraînement uniquement | 🔴 | performance réelle inconnue |
| Étiquetage sexe DI **0,653** contre durée réelle DI environ **1,000** | 🔴 | cible potentiellement biaisée |
| Chemin du modèle codé en dur | 🟠 | déplacement ou changement de machine bloquant l'inférence |
| Sexe encodé manuellement sans justification clinique | 🟠 | biais difficile à expliquer ou contester |
| Colonnes texte supprimées sans pipeline documenté | 🟠 | reproductibilité et maintenance fragilisées |
| Déploiement manuel SSH/scp sans reprise visible | 🟠 | restauration lente en cas d'incident |
| Age : FNR jeunes **0,665**, FPR âgés **0,468** | 🟠 | effet défavorable dépendant de l'usage de la décision |
| Conservation, accès et minimisation des données de santé non documentés | 🟠 | conformité RGPD difficile à démontrer |
| Comparaison des modèles sur un seul split | 🟡 | choix encore exploratoire |
| Modèle **4,73 MB**, inférence **5,1-70,1 ms** | 🟡 | coût compute modéré, énergie et coût opérationnel non mesurés |

## Questions à transmettre au client

- Que signifient concrètement les décisions positive et négative ?
- Le seuil de 7 jours est-il le seuil métier officiel ?
- Comment `sejour_prolonge` est-il défini et appliqué aux groupes ?
- Quelle est la justification clinique de `sexe` et `age` ?
- Une validation humaine, une journalisation et un réexamen sont-ils prévus ?
- Quelle base légale RGPD et quelle qualification AI Act sont retenues ?
