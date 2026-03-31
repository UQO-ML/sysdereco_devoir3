# INF6083 – Projet P3 : Filtrages collaboratif et basé sur la connaissance

**Cours** : INF6083 – Systèmes de recommandation · Hiver 2026  
**Date de remise** : 5 avril 2026

---

## Structure du projet

```
.
├── docker-compose.yml              # Apache Fuseki (triplestore RDF)
├── docker/fuseki/
│   └── recommendations.ttl        # Configuration du dataset Fuseki
├── requirements.txt
├── .env                           # Variables d'environnement (URL Fuseki, etc.)
├── data/
│   ├── raw/                       # Données brutes (MovieLens)
│   └── processed/                 # Résultats, graphiques
├── ontology/
│   ├── recommendation_ontology.ttl # Ontologie OWL
│   └── knowledge_graph_movielens.ttl # Graphe généré (après exécution)
├── notebooks/
│   ├── task0_content_filtering.ipynb
│   ├── task1_collaborative_filtering.ipynb
│   └── task2_knowledge_graph.ipynb
└── src/
    ├── task0_content/
    │   └── content_filtering.py   # TF-IDF + similarité cosinus
    ├── task1_collab/
    │   └── collaborative_filtering.py  # SVD, KNN (scikit-surprise)
    └── task2_knowledge/
        ├── rdf_builder.py         # Construction du graphe RDF
        ├── sparql_queries.py      # Client SPARQL / Fuseki
        └── evaluator.py           # Métriques et comparaison
```

---

## Prérequis

| Outil | Version minimale |
|-------|-----------------|
| Python | 3.11 |
| Docker Desktop | 4.x |
| Docker Compose | v2 |

---

## Installation pas à pas

### 1. Cloner / extraire le projet

```powershell
# Décompresser l'archive ou se placer dans le dossier
cd "INF6083 - Système de recommandation\Devoir 3"
```

### 2. Créer et activer l'environnement virtuel Python

```powershell
# Créer le venv
python -m venv .venv

# Activer (Windows PowerShell)
.\.venv\Scripts\Activate.ps1

# Sur macOS / Linux
# source .venv/bin/activate
```

### 3. Installer les dépendances Python

```powershell
pip install --upgrade pip
pip install -r requirements.txt
```

> **Note** : `scikit-surprise` s'installe via le paquet `surprise==0.1` qui correspond à `scikit-surprise`.  
> Si l'installation échoue sur Windows, installez d'abord Visual C++ Build Tools.

### 4. Télécharger le jeu de données MovieLens 100K

1. Rendez-vous sur https://grouplens.org/datasets/movielens/100k/
2. Téléchargez `ml-latest-small.zip` (recommandé) ou `ml-100k.zip`
3. Extrayez `movies.csv` et `ratings.csv` dans le dossier `data/raw/`

```
data/raw/
├── movies.csv
└── ratings.csv
```

### 5. Créer le fichier env 

Copiez le fichier .env.example en .env et modifiez les valeurs si nécessaire.

### 6. Démarrer Apache Fuseki avec Docker

```powershell
# Lancer Fuseki en arrière-plan
docker compose up -d

# Vérifier que le conteneur est actif
docker compose ps

# Voir les logs
docker compose logs -f fuseki
```

Fuseki sera disponible sur **http://localhost:3030**  
Identifiants : `admin` / `admin123`

#### Créer le dataset manuellement (si non créé automatiquement)

1. Ouvrir http://localhost:3030 dans un navigateur
2. Aller dans **Manage Datasets** → **Add new dataset**
3. Nom : `recommendations`, Type : **Persistent (TDB2)**

#### Valider l'installation avec un graphe RDF minimal

```powershell
# Charger l'ontologie de validation
curl -X PUT http://localhost:3030/recommendations/data \
  -H "Content-Type: text/turtle" \
  -u admin:admin123 \
  --data-binary @ontology/recommendation_ontology.ttl
```

#### Exécuter une requête SPARQL basique de validation

```sparql
PREFIX reco: <http://example.org/reco#>
SELECT ?movie ?title
WHERE {
  ?movie a reco:Movie ;
         reco:title ?title .
}
```

Coller cette requête dans http://localhost:3030/recommendations/sparql

### 7. Enregistrer le kernel Jupyter pour le venv

```powershell
python -m ipykernel install --user --name=inf6083-p3 --display-name "Python 3 (INF6083-P3)"
```

### 8. Lancer Jupyter Notebook

```powershell
jupyter notebook notebooks/
```

### 9. Ordre d'exécution des notebooks

| Ordre | Notebook | Tâche |
|-------|----------|-------|
| 1 | `task0_content_filtering.ipynb` | Filtrage basé sur le contenu |
| 2 | `task1_collaborative_filtering.ipynb` | Filtrage collaboratif |
| 3 | `task2_knowledge_graph.ipynb` | Graphe de connaissances + comparaison |

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

---

## Variables d'environnement (.env)

```
FUSEKI_URL=http://localhost:3030
FUSEKI_DATASET=recommendations
FUSEKI_USER=admin
FUSEKI_PASSWORD=admin123
```

---

## Dépannage

| Problème | Solution |
|----------|----------|
| Port 3030 déjà utilisé | Changer `"3030:3030"` en `"3031:3030"` dans `docker-compose.yml` et mettre à jour `.env` |
| `surprise` ne s'installe pas | `pip install scikit-surprise` |
| Fuseki ne démarre pas | Vérifier que Docker Desktop est lancé, puis `docker compose logs fuseki` |
| `ConnectionRefusedError` sur Fuseki | Attendre ~15s après `docker compose up`, Fuseki met du temps à démarrer |
