# Audit - éthique

## Périmètre et méthode

L'audit porte sur les écarts de prédiction et d'étiquetage selon le sexe, l'âge, les comorbidités et les classes d'IMC. Le disparate impact (DI) est traité comme un signal d'alerte, pas comme une preuve suffisante de discrimination. Les FNR (faux négatifs) et FPR (faux positifs) sont comparés à une référence d'audit provisoire : `dms_jours >= 7`.

## Variables sensibles

- `sexe` est utilisé directement par le modèle via `sexe_bin`, sans justification clinique documentée dans le code.
- `nb_comorbidites`, `imc`, `dms_jours` et `sejour_prolonge` sont des données de santé.
- `age` peut créer des écarts entre groupes et servir de proxy de l'état de santé.
- `departement`, `service` et `type_admission` peuvent être des proxies de pratiques de prise en charge.
- `patient_id` est un identifiant direct et n'est pas utilisé comme feature, ce qui est un point favorable.

## Résultats

### Sexe

Le taux de décisions positives est de **48,6 % chez les hommes** contre **14,1 % chez les femmes**, soit un DI de **0,291** pour les femmes. Le FNR est de **0,680** chez les femmes contre **0,169** chez les hommes, le FPR est de **0,067** chez les femmes contre **0,345** chez les hommes.

L'étiquette historique `sejour_prolonge = 1` concerne **32,1 % des femmes** et **49,1 % des hommes** (DI **0,653**), alors que la référence `dms_jours >= 7` est presque équilibrée (**29,4 %** contre **29,0 %**, DI environ **1,000**). La définition et les règles de production de l'étiquette doivent donc être
clarifiées.

Le groupe réellement désavantagé dépend de l'usage : si la décision positive ouvre un bénéfice, les femmes peuvent en être privées, si elle impose une contrainte, les hommes sont davantage exposés aux faux positifs.

### Age, comorbidités et IMC

Le taux de décisions positives passe de **8,7 %** chez les moins de 40 ans à **58,0 %** chez les 75 ans et plus. Le FNR est de **0,665** chez les moins de 40 ans et le FPR de **0,468** chez les 75 ans et plus. Les étiquettes et la durée réelle progressent dans le même sens avec l'âge, ce résultat est compatible avec une relation clinique, sans exclure un biais.

Le taux de prédictions positives augmente avec les comorbidités, ce qui est cliniquement plausible. Les groupes de 6 et 7 comorbidités sont toutefois très petits (33 et 5 observations). Les classes d'IMC présentent des écarts plus modérés, le DI des prédictions le plus faible est de **0,789** pour la classe 18,5-25.

## Points RGPD et réglementaires à confirmer

Les données de santé appellent une base légale et une exception applicable au regard de l'article 9 du RGPD. La minimisation, les accès, la conservation et la séparation de l'identifiant ne sont pas démontrées dans le dépôt.

La qualification AI Act dépend de la finalité réelle et du rôle du score, le contexte médical ne suffit pas à conclure automatiquement au haut risque.

Pour l'article 22 du RGPD, il faut confirmer l'existence d'une décision exclusivement automatisée et d'un effet juridique ou significatif.

## Questions ouvertes

- Que déclenchent concrètement les décisions positive et négative ?
- Le seuil de 7 jours est-il le seuil métier officiel ?
- Comment `sejour_prolonge` est-il défini et la règle est-elle identique entre groupes ?
- Quelle est la justification clinique de `sexe` et `age` ?
- Existe-t-il une validation humaine, une journalisation et une possibilité de réexamen ?
