# INF6083 – Projet P3 : Filtrages collaboratif et basé sur la connaissance

**Cours** : INF6083 – Systèmes de recommandation · Hiver 2026  
**Date de remise** : 5 avril 2026

---

## Structure du projet

```
.
├── notebook_devoir3.ipynb         # Notebook principal
├── docker-compose.yml             # Apache Fuseki (triplestore RDF)
├── requirements.txt
├── .env                           # Variables d'environnement (URL Fuseki, etc.)
├── data/
│   └── joining/                   
│       └── temporal_pre_split     # Données P2
├── ontology/                      # Ontologie OWL
```

---

## Prérequis


| Outil          | Version          |
| -------------- | ---------------- |
| Python         | 3.11             |
| Docker Desktop | 4.x              |
| Docker Compose | v2               |


---

## Installation pas à pas

### 1. Cloner le projet

```bash
git clone https://github.com/UQO-ML/sysdereco_devoir3.git
```

### 2. Préparer les artefacts réutilisés depuis le projet 2

Récupérer les fichiers suivants depuis le projet P2, dans `data/joining/temporal_pre_split` :

- temporal_pre_split 
- train_interactions.parquet 
- test_interactions.parquet 
- books_representation_sparse.npz 
- user_profiles_tfidf.npz 
- user_ids.npy 
- item_ids.npy 
- item_titles.npy 
- top_n_indices_10.npy

**Le plus rapide étant de créer une archive :**

```bash
cd /sysdereco_devoir2/
tar -czf p3_artifacts_temporal_pre_split.tar.gz \
  -C data/joining/temporal_pre_split \
  train_interactions.parquet \
  test_interactions.parquet \
  books_representation_sparse.npz \
  user_profiles_tfidf.npz \
  user_ids.npy \
  item_ids.npy \
  item_titles.npy \
  top_n_indices_10.npy
```

Et de la décompresser dans `data/joining/temporal_pre_split/` dans ce projet (P3):

```bash
mkdir -p data/joining/temporal_pre_split
tar -xzf p3_artifacts_temporal_pre_split.tar.gz -C data/joining/temporal_pre_split
```

### 3. Créer et activer l'environnement virtuel Python

```powershell
# Créer le venv
python -m venv .venv

# Activer (Windows PowerShell)
.\.venv\Scripts\Activate.ps1

# Sur macOS / Linux
source .venv/bin/activate
```

### 4. Installer les dépendances Python

```powershell
pip install --upgrade pip
pip install -r requirements.txt
```

### 6. Créer le fichier env

Copiez le fichier .env.example en .env et modifiez les valeurs si nécessaire.

```
cp .env.example .env
```

### 7. Démarrer Apache Fuseki avec Docker

```powershell
# Lancer Fuseki en arrière-plan
docker compose up -d

# Vérifier que le conteneur est actif
docker compose ps

# Voir les logs
docker compose logs -f fuseki
```

Fuseki sera disponible sur **[http://localhost:3030](http://localhost:3030)**

---

## Arrêter les services

```powershell
# Arrêter Fuseki
docker compose down

# Arrêter et supprimer les données persistantes
docker compose down -v
```

---

## Aperçu des tâches

### Tâche 0 – Filtrage basé sur le contenu

- Représentation TF-IDF des genres de films
- Similarité cosinus entre items
- Métriques : Precision@10, Recall@10

### Tâche 1 – Filtrage collaboratif

- Algorithmes : SVD (Matrix Factorization), KNN user-based, KNN item-based
- Évaluation : Cross-validation 5-fold (RMSE, MAE)

### Tâche 2 – Graphe de connaissances (RDF + SPARQL)

- Ontologie OWL : classes `User`, `Movie`, `Genre`
- Propriétés : `likesGenre`, `hasGenre`, `mayLike`, `hasInteractedWith`
- Règles d'inférence :
  - **R1** : `User likesGenre G ∧ Movie hasGenre G → User mayLike Movie`
  - **R2** : `User mayLike M1 ∧ M1 hasGenre G ∧ M2 hasGenre G → User mayLike M2`
- Triplestore : Apache Fuseki (Docker)
- Interrogation : SPARQL via SPARQLWrapper

