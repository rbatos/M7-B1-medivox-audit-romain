# Procédure d'audit IA — template 7 sections (MediVox)

> Procédure **fournie** : remplissez chaque section. Un audit **outillé**, pas
> improvisé. Périmètre = observer/documenter/hiérarchiser (≠ corriger, ≠ AIPD).

## 1. Périmètre et hors-périmètre

L’audit porte sur le système MediVox dans son état observé au moment de l’évaluation, avec trois volets principaux :

- **Modèle** : prédicteur DMS qui signale les séjours à risque de prolongation, mis en place par un ancien prestataire de MediVox (modèle présent dans le dossier : `.\legacy\dms_predictor_v1.joblib`). L'audit doit aborder les points suivants : objectif, variables utilisées, données d’entraînement, performances globales et par groupe, seuil de décision, interprétabilité et limites d’usage.
- **Code et architecture** : code présent dans le dossier `.\legacy` dans 2 fichiers sources `predict.py` (script d’inférence de MediVox, il charge le modèle, récupère 4 paramètres en ligne de commande, construit les données d'entrée attendue par le modèle, calcule la probabilité d'un séjour prolongé et affiche la décision avec les probabilités) et `train.py` (entraîne le modèle MediVox en chargeant le dataset, sélectionne les features, entraîne une forêt aléatoire pour la prédiction, sauvegarde le modèle puis affiche son exactitude). L'audit doit aborder les points suivants : pipeline de préparation et d’inférence, validation des entrées, gestion des erreurs, dépendances, secrets, flux de données, déploiement et points de rupture.
- **Dataset** : fichier `csv` de données (dans le dossier `.\data\dms_dataset.csv`) qui contient 9 features (dont une donnant la cible si on fixe un seuil `dms_jours`) et 1 cible. L'audit doit aborder les points suivants : provenance, représentativité, qualité, valeurs manquantes, variables sensibles ou indirectement sensibles, étiquetage, déséquilibres et conditions de conservation.

L’audit pourrait également inclure l’examen de la documentation disponible.

Sont exclus du périmètre, selon les trois volets audités :

- **Modèle** : la refonte du modèle et l’évaluation clinique définitive du dispositif
- **Code et architecture** : le **pen-test**, l’audit offensif de cybersécurité et la refonte du code ou de l’architecture
- **Dataset** : la refonte du dataset.

Les éléments suivants sont également exclus, car ils sont transversaux aux trois volets :

- la réalisation formelle d’une **AIPD** ou d’une analyse juridique complète
- la certification ou la déclaration de conformité réglementaire.

## 2. Audit éthique

### Variables sensibles et indirectement sensibles

- **`sexe`** est une variable sensible au regard de l'analyse de discrimination : elle est utilisée directement par le modèle sous la forme `sexe_bin`, sans justification clinique documentée dans le code.
- **`nb_comorbidites`** et **`imc`** sont des données relatives à la santé. Elles sont utilisées comme variables prédictives et doivent être justifiées, limitées à la finalité annoncée et protégées.
- **`age`** est une donnée personnelle qui peut créer des écarts entre classes et agir comme proxy de l'état de santé.
- **`departement`**, **`service`** et **`type_admission`** peuvent être des variables indirectement sensibles ou des proxies de pratiques de prise en charge. Elles ne sont pas utilisées par le modèle actuel, mais leur effet sur les étiquettes et la durée réelle doit être vérifié.
- **`patient_id`** est un identifiant direct à protéger. Il n'est pas utilisé dans les features du modèle, ce qui constitue un point positif.
- **`dms_jours`** et **`sejour_prolonge`** sont des informations liées au séjour et à la santé. La première sert ici à construire une référence d'audit, la seconde constitue la cible historique du modèle.

### Disparate impact et investigation

Le disparate impact est traité comme un **signal d'alerte**, et non comme une preuve suffisante de discrimination.

- Pour le **sexe**, le taux de prédictions positives est de **48,6 % chez les hommes** contre **14,1 % chez les femmes**, soit un DI de **0,291** pour les femmes en prenant les hommes comme référence. Les femmes ont davantage de faux négatifs (FNR **0,680** contre **0,169** chez les hommes), tandis que les hommes ont davantage de faux positifs (FPR **0,345** contre **0,067** chez les femmes). Le groupe effectivement désavantagé dépend donc de l'usage : si la décision positive ouvre un bénéfice, les femmes risquent d'en être privées. Si elle entraîne une contrainte, les hommes sont davantage exposés aux faux positifs.
- L'investigation de la cible montre que les étiquettes historiques `sejour_prolonge = 1` concernent **32,1 % des femmes** et **49,1 % des hommes** (DI **0,653**), alors que la référence de durée `dms_jours >= 7` est presque identique (**29,4 %** contre **29,0 %**, DI environ **1,000**). La différence d'étiquetage ne s'explique donc pas simplement par la durée réelle. la définition de la cible et les règles historiques d'étiquetage doivent être clarifiées.
- Pour l'**âge**, le taux de prédictions positives passe de **8,7 %** chez les moins de 40 ans à **58,0 %** chez les 75 ans et plus (DI **0,150** pour les moins de 40 ans). Les moins de 40 ans ont un FNR de **0,665**, tandis que les 75 ans et plus ont un FPR de **0,468**. Les plus jeunes sont donc davantage exposés aux non-détections et les plus âgés aux signalements à tort. Le préjudice dépend là encore de la conséquence associée à la décision positive. Les étiquettes et la durée réelle progressent toutefois dans le même sens avec l'âge. Ce signal est donc compatible avec une différence clinique, sans suffire à exclure un biais.
- Pour les **comorbidités**, la progression du taux de prédictions positives est cohérente avec l'hypothèse qu'un état de santé plus chargé augmente le risque de séjour prolongé. Les groupes 6 et 7 comorbidités sont néanmoins très petits (33 et 5 observations), ce qui limite la robustesse de leurs indicateurs.

L'usage réel détermine la gravité du préjudice : ces écarts deviennent critiques si la sortie `RISQUE_SEJOUR_PROLONGE` sert à orienter automatiquement la surveillance, les ressources ou la prise en charge. Le script `predict.py` produit une décision binaire à partir d'un seuil de 0,5, sans supervision humaine, journalisation ni traçabilité. Il faut confirmer auprès de MediVox si cette sortie est seulement informative ou si elle déclenche effectivement une action sur le patient.

### RGPD, AI Act et article 22

- **RGPD, article 9** : les comorbidités, l'IMC, la durée de séjour et la cible concernent la santé. Leur traitement appelle une base juridique et une exception applicable aux données de santé, qui ne peuvent pas être déduites du seul dépôt de code. L'audit vérifie aussi la minimisation : seules les variables nécessaires à la finalité doivent être utilisées, avec une durée de conservation, des accès et une séparation entre identifiant et variables d'apprentissage documentés. Ces éléments ne sont pas démontrés dans le dépôt.
- **AI Act, article 6** : le domaine médical ne suffit pas, à lui seul, à qualifier automatiquement le système de haut risque. La qualification dépend de la finalité prévue, du rôle exact du score dans la décision et de son rattachement éventuel à une catégorie de l'annexe III. Si le système est qualifié de haut risque, les obligations correspondantes doivent être examinées, notamment la gestion des risques, la qualité des données, la traçabilité, la supervision humaine, la robustesse et la cybersécurité. La qualification reste à confirmer avec MediVox.
- **RGPD, article 22 - première condition** : le score semble produire une décision automatisée, puisque `predict.py` applique directement le seuil de 0,5. Il faut toutefois confirmer si une intervention humaine réelle et significative existe en pratique.
- **RGPD, article 22 - seconde condition** : l'effet juridique ou l'effet significatif sur la personne n'est pas établi par le dépôt. Il serait présent si le score conditionnait ou influençait fortement l'accès à une prise en charge, la priorisation des soins ou les ressources. L'usage opérationnel et les conséquences pour le patient doivent donc être documentés avant toute conclusion juridique.

Questions prioritaires à poser à MediVox : 
- quelle est la justification clinique de `sexe` et de `age` ?
- Comment `sejour_prolonge` est-il défini et le seuil de 7 jours est-il le seuil métier officiel ?
- Le score déclenche-t-il une décision sur le patient, et existe-t-il une validation humaine, une journalisation et une possibilité de contestation ?

## 3. Audit technique

### Architecture, modularité et couplage

L'architecture observée repose sur deux scripts Python historiques et un fichier de modèle `joblib` : `train.py` prépare les données et entraîne le modèle, puis `predict.py` recharge ce fichier et réalise l'inférence. Le fonctionnement est compréhensible et la chaîne entraînement-prédiction est cohérente sur les features utilisées (`age`, `nb_comorbidites`, `imc`, `sexe_bin`).

En revanche, les responsabilités sont peu modularisées : la préparation des features est écrite directement dans `train.py` et reconstruite séparément dans `predict.py`, sans pipeline partagé ni schéma versionné. Cette duplication crée un couplage implicite entre l'ordre des colonnes, l'encodage manuel du sexe et la structure attendue par le modèle. Une modification d'un script ou du dataset peut donc rendre l'autre incompatible sans détection automatique.

Le modèle est aussi chargé depuis un fichier local codé en dur dans `predict.py`. Le chemin dépend du répertoire courant d'exécution : un déplacement du fichier, une nouvelle machine ou un lancement depuis un autre répertoire peut interrompre l'inférence. Le fichier `joblib` ne contient pas, d'après le dépôt observé, de métadonnées explicites sur la version du dataset, les features, le seuil, les dépendances ou la date d'entraînement.

### Sécurité, validation et transport

- **Secrets** : `train.py` contient un mot de passe de production en clair (`DB_PASSWORD`). Il doit être considéré comme exposé et sa présence empêche de considérer le dépôt comme sûr pour un déploiement tel quel.
- **Validation** : `predict.py` convertit directement les arguments en `int` et `float`, sans schéma, bornes métier, contrôle des valeurs manquantes ni gestion d'erreur. Une valeur incohérente peut provoquer un arrêt du script ou une prédiction non fiable.
- **Préparation des données** : `train.py` supprime les colonnes texte au lieu d'utiliser un pipeline d'encodage documenté. Le sexe est encodé manuellement en 0/1 et utilisé comme feature sensible, sans justification clinique visible ni contrôle de son impact.
- **Transport et déploiement** : les commentaires du script indiquent un appel en production via SSH et un déploiement manuel par `scp`. Le dépôt ne montre pas de chiffrement applicatif, de gestion de certificats, de contrôle d'intégrité du modèle, de gestion des versions ou de procédure de rotation des secrets. Le périmètre ne comprend pas un pen-test, mais ces éléments peuvent être constatés comme risques d'architecture et de déploiement.
- **Traçabilité** : `predict.py` affiche une décision mais ne journalise ni l'identifiant de version du modèle, ni l'entrée, ni la sortie, ni l'utilisateur ou le contexte de décision. Dans un contexte médical, l'absence de log rend l'incident ou la décision difficile à reconstituer.

### Scalabilité et points de rupture

Les mesures du notebook sont modestes sur les volumes testés : **5,1 ms** pour 100 lignes, **13,2 ms** pour 1 000 et **70,1 ms** pour 10 000. Le modèle pèse **4,73 MB**. Cela suggère une exécution peu coûteuse, mais ne remplace pas un test de charge avec concurrence et pics de trafic.

Les principaux points de rupture sont les suivants :

- le fichier modèle local et la machine qui l'héberge constituent un **SPOF** : leur indisponibilité bloque les prédictions.
- le déploiement manuel par SSH/scp dépend d'une procédure et d'un accès individuels, sans mécanisme visible de reprise ou de retour arrière.
- l'absence de validation et de gestion d'erreur peut interrompre le flux sur une seule entrée invalide.
- l'absence de journalisation et de supervision empêche de détecter rapidement une panne, une dérive des entrées ou une dégradation des sorties.
- les mesures réalisées ne couvrent ni plusieurs requêtes concurrentes, ni la disponibilité réseau, ni le temps de chargement du modèle, ni les pics de charge.

**Conclusion technique :** le modèle est peu coûteux à exécuter, mais le système reste fortement dépendant d'une machine, d'un fichier local et de scripts non modulaires. Le risque principal ne vient donc pas de la capacité de calcul mesurée, mais de la fragilité du déploiement, de la validation des entrées, de la gestion des secrets et de l'absence de traçabilité.

## 4. Audit ressources

### Mesures du modèle historique

Le modèle Random Forest historique occupe **4,73 MB** sur disque. Dans l'environnement d'audit', le temps d'inférence est de **5,1 ms pour 100 lignes**, **13,2 ms pour 1 000** et **70,1 ms pour 10 000**. La RSS du processus est de **186,62 MB** avant inférence et augmente d'environ **0,49 MB** après le test à 10 000 lignes.

Ces mesures indiquent un coût d'inférence et une empreinte modèle modestes sur les volumes testés. Elles ne constituent toutefois pas une mesure de production : la concurrence, les pics de charge, le chargement du modèle et le réseau n'ont pas été testés. Le temps d'entraînement du `train.py` historique n'a pas été
mesuré séparément lors de l'audit.

### Comparaison à deux alternatives

Les trois modèles ont été entraînés et évalués sur le même découpage train/test :

| Modèle | ROC-AUC test | Temps d'entraînement | Temps d'inférence test | Taille sérialisée |
|---|---:|---:|---:|---:|
| Random Forest historique | 0,7195 | 362,1 ms | 13,9 ms | 4,37 MB |
| Régression logistique | **0,7364** | **6,9 ms** | **2,2 ms** | **0,0013 MB** |
| Gradient Boosting | 0,7199 | 1 004,9 ms | 15,7 ms | 0,34 MB |

Sur ce test, la régression logistique est à la fois la plus légère et la plus rapide, avec un ROC-AUC légèrement supérieur au Random Forest. Le Gradient Boosting est plus coûteux sans gain de performance visible. Cette comparaison reste exploratoire : elle repose sur une seule séparation train/test et ne remplace pas une validation croisée ni une comparaison des résultats par groupe.

### Lecture sobriété

Le modèle historique est peu coûteux à stocker et à exécuter, mais la régression logistique réduit encore les ressources mesurées : environ **3 600 fois moins de stockage** que le Random Forest et un temps d'entraînement environ **52 fois inférieur** dans cette exécution. Ces chiffres appuient un argument de sobriété sur le coût de calcul et de stockage, sans permettre d'estimer directement l'énergie consommée, les émissions carbone ou le coût opérationnel complet. Ce dernier inclut aussi le déploiement, la supervision, la maintenance, la validation humaine et la traçabilité.

## 5. Tableau d'indicateurs consolidé

| Indicateur | Sévérité | Conséquence client |
|---|---|---|
| Mot de passe de production écrit en clair dans `train.py` | 🔴 | compromission potentielle de la production et nécessité de révoquer le secret |
| Décision automatique sans supervision humaine, journalisation ni traçabilité dans un contexte médical | 🔴 | décision contestable, impossible à reconstituer et risque accru pour le patient |
| Écart entre les groupes de sexe : DI des prédictions de **0,291**, FNR de **0,680** chez les femmes et FPR de **0,345** chez les hommes | 🔴 | femmes potentiellement privées d'un bénéfice si la décision positive aide à la prise en charge, hommes davantage exposés aux contraintes si elle est défavorable |
| Absence de validation des entrées, de schéma et de gestion d'erreur dans `predict.py` | 🔴 | entrée incohérente susceptible de provoquer une panne ou une prédiction non fiable |
| Fichier modèle local et machine d'exécution constituant un **SPOF** | 🔴 | indisponibilité du modèle pouvant bloquer toutes les prédictions |
| Accuracy mesurée uniquement sur les données d'entraînement, sans split train/test ni validation | 🔴 | performance réelle inconnue et risque de décisions médicales insuffisamment fiables |
| Étiquetage historique différent selon le sexe (DI **0,653**) alors que la référence `dms_jours >= 7` est presque équilibrée (DI environ **1,000**) | 🔴 | cible potentiellement biaisée et apprentissage d'une différence qui ne correspond pas à la durée réelle |
| Chemin du modèle codé en dur et dépendant du répertoire courant dans `predict.py` | 🟠 | déplacement du fichier ou changement de machine pouvant interrompre l'inférence |
| Sexe encodé manuellement en 0/1 et utilisé comme feature sans justification clinique documentée | 🟠 | biais ou discrimination difficiles à détecter, expliquer et contester |
| Colonnes texte supprimées sans pipeline ni encodage documenté dans `train.py` | 🟠 | perte d'information et préparation des données difficile à reproduire ou maintenir |
| Déploiement manuel par SSH/scp sans versionnement, contrôle d'intégrité ni retour arrière visible | 🟠 | erreurs de déploiement et restauration lente en cas d'incident |
| Écarts entre classes d'âge : FNR de **0,665** chez les moins de 40 ans et FPR de **0,468** chez les 75 ans et plus | 🟠 | effet défavorable variable selon que la décision positive ouvre un bénéfice ou impose une contrainte |
| Données de santé, conservation, accès et minimisation non documentés dans le dépôt | 🟠 | difficulté à démontrer la maîtrise des données au regard du RGPD, article 9 |
| Comparaison des modèles fondée sur une seule séparation train/test | 🟡 | choix de modèle encore exploratoire, non confirmé par validation croisée ni analyse par groupe |
| Coût de calcul mesuré comme modéré : modèle de **4,73 MB** et inférence de **5,1 à 70,1 ms** sur les volumes testés | 🟡 | point favorable pour la sobriété, mais absence de mesure directe de l'énergie et du coût opérationnel complet |

## 6. Synthèse exécutive

Le disparate impact révèle des écarts entre les sexes, mais le groupe lésé dépend de l'usage réel du score. Les femmes ont davantage de faux négatifs, elles peuvent donc être privées d'un bénéfice si la décision positive déclenche une aide ou une priorisation. Les hommes ont davantage de faux positifs, ils peuvent être davantage exposés si cette décision entraîne une contrainte ou un désavantage. La conséquence réelle doit être confirmée avec MediVox.

Ce résultat constitue un **signal d'alerte**, et non une preuve suffisante de discrimination. Les étiquettes historiques sont différentes selon le sexe (DI **0,653**), alors que la durée réelle d'au moins 7 jours est presque équivalente entre les deux groupes. La définition de `sejour_prolonge`, le seuil métier de 7 jours et la justification clinique de l'utilisation du sexe et de l'âge doivent donc être confirmés auprès de MediVox. Les écarts observés selon l'âge doivent également être surveillés : les moins de 40 ans ont davantage de faux négatifs, tandis que les 75 ans et plus ont davantage de faux positifs.

Plusieurs fragilités techniques augmentent le risque d'usage : mot de passe de production présent en clair dans `train.py`, absence de validation des entrées et de gestion d'erreur dans `predict.py`, décision sans journalisation ni traçabilité, et dépendance à une machine et à un fichier modèle local qui constituent un point de rupture unique. Les performances historiques n'ont par ailleurs pas été validées sur un jeu de test indépendant.

Le coût de calcul observé reste modéré : le modèle pèse environ **4,73 MB** et l'inférence prend **5,1 à 70,1 ms** pour 100 à 10 000 lignes. Une régression logistique obtient sur le test réalisé un ROC-AUC de **0,7364**, contre **0,7195** pour le Random Forest, avec une taille et un temps d'entraînement nettement inférieurs. Cette comparaison reste exploratoire et ne permet pas encore de choisir un modèle pour la production.

**Conclusion pour la décision :** avant toute utilisation opérationnelle, il faut clarifier la finalité réelle du score, les règles d'étiquetage, la justification des variables sensibles, le niveau de supervision humaine et les obligations réglementaires applicables. L'audit ne propose pas de nouvelle architecture, il hiérarchise les risques et les questions auxquelles MediVox doit répondre.


## 7. Questions ouvertes

- Que signifie concrètement une décision positive ou négative pour le patient : bénéfice, priorisation, contrainte ou simple information ?
- Quel est le seuil métier officiel d’un séjour prolongé, et comment `sejour_prolonge` a-t-il été construit ?
- Quelle est la justification clinique documentée de l'utilisation de `sexe` et de `age` ?
- Les écarts de FNR et de FPR sont-ils acceptables au regard de l'usage réel du score ?
- Une validation humaine réelle existe-t-elle avant toute action, et le patient peut-il contester ou faire réexaminer la décision ?
- Les décisions, les entrées, la version du modèle et les suites données sont-elles journalisées et conservées pendant une durée définie ?
- Quelle qualification AI Act et quelle base légale RGPD MediVox retient-elle, sous réserve de validation juridique ?
