# Rapport d'audit — prédicteur de séjour prolongé v1 (MediVox)

## 1. Synthèse exécutive (5 min)

### Point le plus important

⚖️ L'audit révèle un écart important entre les groupes de sexe : le modèle produit une décision positive pour **48,6 % des hommes** contre **14,1 % des femmes** (disparate impact, ou DI : rapport entre le taux d'un groupe et celui du groupe de référence, de **0,291** pour les femmes). Les erreurs sont asymétriques : les femmes ont davantage de faux négatifs (FNR **0,680** contre **0,169** chez les hommes), tandis que les hommes ont davantage de faux positifs (FPR **0,345** contre **0,067** chez les femmes).

Le groupe effectivement désavantagé dépend toutefois de l'usage réel. Si la décision positive ouvre un bénéfice ou une priorisation, les femmes peuvent en être privées. Si elle entraîne une contrainte ou un désavantage, les hommes peuvent être davantage exposés aux faux positifs. MediVox doit donc préciser ce que déclenchent concrètement les décisions `RISQUE_SEJOUR_PROLONGE` et `SEJOUR_STANDARD`.

⚖️ Ce DI est un signal d'alerte, pas une preuve suffisante de discrimination. Les étiquettes historiques diffèrent selon le sexe (DI **0,653**), alors que la référence de durée `dms_jours >= 7` est presque équilibrée (**29,4 %** chez les femmes contre **29,0 %** chez les hommes). La définition de `sejour_prolonge`, le seuil métier de 7 jours et la justification clinique de `sexe` et `age` sont à confirmer.

👩‍💻 Le système présente aussi des risques opérationnels : secret de production en clair, absence de validation des entrées, absence de journalisation et dépendance à une machine et à un fichier modèle local. La performance historique n'a pas été évaluée sur un jeu de test indépendant.

Le coût de calcul observé reste modéré : modèle d'environ **4,73 MB** et inférence de **5,1 à 70,1 ms** pour 100 à 10 000 lignes. Une régression logistique obtient un ROC-AUC de **0,7364**, contre **0,7195** pour le Random Forest historique, mais cette comparaison reste exploratoire. La décision avant usage opérationnel dépend donc de la clarification de la finalité, des conséquences des décisions, de la supervision humaine et des obligations réglementaires applicables.

## 2. Contexte et périmètre 👩‍💻 ⚖️

MediVox utilise un prédicteur de séjour prolongé fourni par un ancien prestataire. L'audit porte sur le modèle `legacy/dms_predictor_v1.joblib`, les scripts `legacy/train.py` et `legacy/predict.py`, ainsi que le dataset
`data/dms_dataset.csv`.

Le modèle utilise `age`, `nb_comorbidites`, `imc` et `sexe_bin` pour prédire la cible historique `sejour_prolonge`. L'audit couvre les données, les performances par groupe, le code, la sécurité observable, le déploiement, les ressources et l'usage réel du score.

Sont hors périmètre le pen-test, l'AIPD formelle, l'analyse juridique complète, la refonte du modèle, du code ou des données, la certification et l'évaluation clinique définitive. Les conclusions réglementaires restent donc à confirmer avec MediVox et les fonctions compétentes.

## 3. Volet éthique ⚖️

### Variables et données

`sexe` est utilisé directement comme variable de modèle sous la forme `sexe_bin`, sans justification clinique documentée. `nb_comorbidites`, `imc`, `dms_jours` et `sejour_prolonge` sont des données relatives à la santé.
`age` peut créer des écarts entre classes et servir de proxy de l'état de santé. `departement`, `service` et `type_admission` peuvent être des variables indirectement sensibles ou des proxies de pratiques de prise en charge. Ils ne sont pas utilisés par le modèle actuel. `patient_id` est un identifiant direct et n'est pas utilisé comme feature, ce qui est un point favorable.

### Résultats et investigation

Le DI est interprété comme un signal à investiguer. Pour le sexe, les femmes ont davantage de faux négatifs et les hommes davantage de faux positifs. Le préjudice ne peut être attribué à un groupe sans connaître la conséquence de la décision positive.

Pour l'âge, les prédictions positives passent de **8,7 %** chez les moins de 40 ans à **58,0 %** chez les 75 ans et plus. Le FNR est de **0,665** chez les moins de 40 ans et le FPR de **0,468** chez les 75 ans et plus. Les étiquettes et la durée réelle progressent dans le même sens avec l'âge, ce qui est compatible avec une relation clinique, sans exclure un biais.

Pour les comorbidités, la hausse du taux de prédictions positives est cliniquement plausible. Les groupes de 6 et 7 comorbidités sont cependant très petits (33 et 5 observations).

### RGPD, AI Act et article 22

⚖️ Les données de santé appellent une base légale et une exception applicable au regard de l'article 9 du RGPD, elles ne sont pas démontrées par le dépôt. La minimisation, les accès, la conservation et la séparation de l'identifiant doivent être documentées.

La qualification AI Act ne découle pas du seul contexte médical. Elle dépend de la finalité réelle, du rôle du score dans la décision et de son rattachement éventuel à une catégorie de l'annexe III. Si le système est haut risque, les obligations correspondantes devront être examinées.

Pour l'article 22 du RGPD, deux conditions sont à vérifier :

1. le score produit-il une décision exclusivement automatisée, sans intervention humaine réelle et significative ?
2. cette décision produit-elle un effet juridique ou un effet significatif sur la personne, par exemple sur la priorisation des soins ou des ressources ?

## 4. Volet technique 👩‍💻

L'architecture est lisible mais peu modulaire : `train.py` et `predict.py` reconstruisent séparément les features, sans pipeline partagé ni schéma versionné. Cela crée un couplage entre l'ordre des colonnes, l'encodage du sexe et le modèle. Le chemin du fichier modèle est codé en dur et dépend du répertoire courant.

`train.py` contient un mot de passe de production en clair. `predict.py` convertit directement les arguments en `int` et `float`, sans validation de schéma, de bornes ou de valeurs manquantes, ni gestion d'erreur. Le code indique un déploiement manuel par SSH/scp, sans versionnement, contrôle d'intégrité ou journalisation démontrés.

Le fichier modèle et la machine qui l'héberge constituent un point de rupture unique (SPOF : une panne suffit à interrompre le service). Une entrée invalide, une erreur de déploiement ou l'absence de supervision peut aussi interrompre ou retarder la détection d'un problème.

## 5. Volet ressources 👩‍💻

Le modèle historique pèse **4,73 MB**. L'inférence mesurée prend **5,1 ms** pour 100 lignes, **13,2 ms** pour 1 000 et **70,1 ms** pour 10 000. La RSS du processus est de **186,62 MB** avant inférence et augmente d'environ **0,49 MB** après le test à 10 000 lignes.

Sur un même découpage train/test, la régression logistique obtient le meilleur ROC-AUC (**0,7364**) et la plus petite taille sérialisée (**0,0013 MB**). Le Random Forest obtient **0,7195** et le Gradient Boosting **0,7199**. La régression logistique est aussi la plus rapide à entraîner (**6,9 ms**) et à inférer (**2,2 ms**).

Ces chiffres appuient un constat de sobriété sur le stockage et le calcul, mais ne mesurent ni l'énergie, ni les émissions carbone, ni le coût opérationnel complet. La comparaison reste exploratoire : une seule séparation train/test a été utilisée.

## 6. Tableau consolidé des risques 🔴 🟠 🟡

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
| Âge : FNR jeunes **0,665**, FPR âgés **0,468** | 🟠 | effet défavorable dépendant de l'usage de la décision |
| Conservation, accès et minimisation des données de santé non documentés | 🟠 | conformité RGPD difficile à démontrer |
| Comparaison des modèles sur un seul split | 🟡 | choix encore exploratoire |
| Modèle **4,73 MB**, inférence **5,1-70,1 ms** | 🟡 | coût compute modéré, énergie et coût opérationnel non mesurés |

## 7. Questions ouvertes pour le client ⚖️ 👩‍💻

- Que signifie concrètement une décision positive ou négative : bénéfice, priorisation, contrainte ou simple information ?
- Le seuil de 7 jours est-il le seuil métier officiel et `sejour_prolonge` est-il défini directement à partir de `dms_jours` ?
- Quelle est la justification clinique documentée de `sexe` et `age` ?
- Les écarts de FNR et de FPR sont-ils acceptables pour l'usage réel du score ?
- Une validation humaine réelle existe-t-elle avant toute action sur le patient ?
- Les décisions, les entrées, la version du modèle et les suites données sont-elles journalisées et conservées pendant une durée définie ?
- Quelle base légale RGPD et quelle qualification AI Act MediVox retient-elle, sous réserve de validation juridique ?
- Qui est responsable de la décision finale et existe-t-il une possibilité de contestation ou de réexamen ?

## 8. Définitions et repères de lecture ⚖️ 👩‍💻

- **Feature** : variable fournie au modèle pour produire une prédiction, par exemple `age`, `imc` ou `nb_comorbidites`.
- **Disparate impact (DI)** : rapport entre le taux de décisions positives d'un groupe et celui du groupe de référence. Un DI de **0,291** signifie que le premier groupe reçoit environ 29,1 % du taux de décisions positives du groupe de référence.
- **Faux négatif (FN)** : séjour réellement prolongé mais non signalé par le modèle. **FNR** (*False Negative Rate*) : proportion de faux négatifs parmi les séjours réellement prolongés.
- **Faux positif (FP)** : séjour signalé comme prolongé alors qu'il ne l'est pas selon la référence utilisée. **FPR** (*False Positive Rate*) : proportion de faux positifs parmi les séjours non prolongés.
- **ROC-AUC** : mesure de la capacité d'un modèle à distinguer les deux classes, ici séjour prolongé et séjour standard. Plus la valeur est élevée, meilleure est la séparation observée, elle ne prouve pas à elle seule l'équité ou la pertinence clinique.
- **RSS** (*Resident Set Size*) : quantité de mémoire vive effectivement utilisée par le processus au moment de la mesure.
- **SPOF** (*Single Point of Failure*) : point de défaillance unique. Sa panne suffit à interrompre le service, comme la machine ou le fichier modèle local.
- **Proxy** : variable qui peut indirectement représenter une autre variable, par exemple un service ou un département pouvant refléter des pratiques de prise en charge.
- **Calibration** : adéquation entre une probabilité annoncée et la fréquence réellement observée. Une probabilité de 0,70 devrait correspondre, sur un ensemble comparable, à environ 70 % de cas positifs.
- **AIPD** : analyse d'impact relative à la protection des données, réalisée lorsqu'un traitement est susceptible d'engendrer un risque élevé pour les personnes.
- **AI Act** : règlement européen sur l'intelligence artificielle. Dans ce rapport, la qualification et les obligations éventuelles restent à confirmer selon la finalité et l'usage réel du système.
- **Article 22 du RGPD** : disposition relative aux décisions fondées exclusivement sur un traitement automatisé lorsqu'elles produisent un effet juridique ou un effet significatif sur une personne.

