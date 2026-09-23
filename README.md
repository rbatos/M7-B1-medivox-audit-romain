# M7-B1 — Auditer une architecture IA héritée (MediVox Cliniques)

> **Repo template.** « Use this template » → `M7-B1-medivox-audit-<prenom>`.

## 🚀 Démarrage

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pytest -q tests               # l'environnement d'audit fonctionne (3 tests verts)
python legacy/train.py        # le modèle à auditer (déjà fourni, regénérable)
jupyter notebook notebooks/M7-B1_template.ipynb
```

> Variante `uv` : `uv venv .venv && source .venv/bin/activate` puis
> `uv pip install -r requirements.txt`.
> Dépannage : `No module named pip` → vous êtes dans un venv créé par `uv`,
> utilisez `uv pip install …` (pas `pip install`).

**Fourni** : `legacy/` (code héritage à auditer), `data/dms_dataset.csv` (10k séjours), `procedure_audit.md` (template 7 sections).

## 🧩 Comment lire les livrables

Les fichiers suivent une logique simple :

| Élément | Rôle |
|---|---|
| `procedure_audit.md` | La checklist de l'audit : elle fixe le périmètre, les questions à traiter et les éléments à vérifier. |
| `audit/0X_*.md` | Le dossier de preuves : chaque volet rassemble les constats, les mesures, les limites et les éléments qui justifient l'analyse. |
| `rapport_audit.md` | La décision client : il synthétise les risques prioritaires, les questions restantes et les points à confirmer par MediVox. |

En pratique, on suit la procédure, on documente les preuves dans `audit/`, puis on s'appuie sur ces preuves pour rédiger le rapport destiné aux décideurs.

## ⚡ Parcours express de relecture

Depuis la racine du dépôt, suivre cet ordre pour retrouver rapidement le périmètre, les preuves et les décisions à prendre :

| Temps | Lecture | Question à retenir |
|---|---|---|
| 0:00–1:00 | `audit/04_consolidation.md` | Quels sont les risques 🔴/🟠 prioritaires ? |
| 1:00–2:00 | `audit/01_ethique.md` | Quels groupes sont exposés et que déclenche réellement le score ? |
| 2:00–3:00 | `audit/02_technique.md` | Quelles fragilités du code et du déploiement rendent le système peu maîtrisable ? |
| 3:00–4:00 | `audit/03_ressources.md` | Quelles mesures sont observées, et quelles limites ont-elles ? |
| 4:00–5:00 | `rapport_audit.md` puis les questions ouvertes de `procedure_audit.md` | Qu'est-ce qui doit être confirmé par MediVox avant toute décision ? |

Pour vérifier un chiffre, ouvrir ensuite `notebooks/M7-B1.ipynb` : il calcule les écarts par groupe, la référence `dms_jours >= 7`, les mesures `psutil` et la comparaison des modèles.

## 🔁 Reproduire les mesures

Après l'installation des dépendances, une seule commande suffit depuis la
racine du dépôt :

```bash
jupyter nbconvert --to notebook --execute notebooks/M7-B1.ipynb --output M7-B1-reproduit.ipynb --output-dir notebooks
```

Le notebook charge directement `data/dms_dataset.csv` et le modèle fourni `legacy/dms_predictor_v1.joblib`. Il exécute les calculs d'écarts par groupe, les mesures de ressources et la comparaison des modèles, puis écrit une copie avec les sorties dans `notebooks/M7-B1-reproduit.ipynb`.

Les commandes suivantes sont utiles pour contrôler ou régénérer l'environnement, mais ne sont pas nécessaires pour reproduire les mesures :

```bash
pytest -q tests                  # vérifie les données, le modèle et l'inférence
python legacy/train.py           # régénère le modèle fourni si nécessaire
```

Les temps, la mémoire et les tailles observés dépendent de la machine et de l'environnement. Ils doivent être comparés aux résultats documentés, pas considérés comme des valeurs de production.

## 🧭 Ce qui a été produit

| Livrable produit | Fichier | Contenu |
|---|---|---|
| Procédure d'audit renseignée | `procedure_audit.md` | Périmètre, contrôles attendus, constats, limites et questions ouvertes. |
| Audit éthique et réglementaire | `audit/01_ethique.md` | Biais par groupe, disparate impact, FNR/FPR, RGPD, AI Act et article 22. |
| Audit technique | `audit/02_technique.md` | Architecture, code historique, sécurité observable, déploiement et points de rupture. |
| Audit ressources et sobriété | `audit/03_ressources.md` | Mesures de temps, mémoire et taille du modèle, puis comparaison des modèles. |
| Consolidation des risques | `audit/04_consolidation.md` | Tableau des risques hiérarchisés avec sévérité et conséquence client. |
| Notebook de mesures | `notebooks/M7-B1.ipynb` | Calculs reproductibles des écarts, des performances et des ressources. |
| Rapport d'audit client | `rapport_audit.md` | Synthèse exécutive, décision à instruire, risques prioritaires et questions à confirmer. |

## ✅ Réussite

- **Disparate impact calculé** sur ≥ 1 variable sensible, **puis investigué** : préjudice défini, erreurs (FNR/FPR) par groupe, étiquette confrontée à `dms_jours`.
- **Qualification AI Act raisonnée** (art. 6, usage réel décrit) et article 22 du RGPD examiné sur ses 2 conditions — pas de « santé = haut risque » présumé.
- Mesures psutil **chiffrées** et comparées à ≥ 1 alternative.
- Tableau ≥ 12 lignes en 🔴/🟠/🟡.
- Le rapport **hiérarchise et questionne** (ne propose pas la solution — c'est M7-B2).
- Le rapport croise les enjeux techniques, éthiques et réglementaires et formule les questions à confirmer par MediVox.
- Aucun journal de bord séparé ni deux grilles de lecture explicitement attribuées à Hélène et Marc ne sont présents dans les livrables actuels.

## 📚 Ressources

Voir [`./ressources/`](./ressources/) — 5 mini-cours + `liens_officiels.md`.
