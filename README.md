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
│   └── temporal_pre_split         # Données P2
├── docker/
│   └── Dockerfile
├── fuseki_config.ttl              # Configuration Fuseki
├── ontology/
│   ├── sysdereco.owl              # Ontologie OWL de base (2.1)
│   ├── sysdereco_inferred.owl     # Ontologie OWL inferrée (2.2)
│   └── sysdereco.ttl              # Triplets RDF
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

Récupérer les fichiers suivants depuis le projet P2, dans `data/temporal_pre_split` :

- temporal_pre_split 
- train_interactions.parquet 
- test_interactions.parquet 
- books_representation_sparse.npz 
- user_profiles_tfidf.npz 
- user_ids.npy 
- item_ids.npy 
- item_titles.npy 
- top_n_indices_10.npy


### 3. Créer et activer l'environnement virtuel Python

```bash
# Créer le venv
python -m venv .venv

# Activer (Windows PowerShell)
.\.venv\Scripts\Activate.ps1

# Sur macOS / Linux(dependant du shell utilisé)
source .venv/bin/activate
```

### 4. Installer les dépendances Python

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

### 6. Créer le fichier env

Copiez le fichier .env.example en .env et modifiez les valeurs si nécessaire.

```
cp .env.example .env
```

### 7. Démarrer Apache Fuseki avec Docker

```bash
# Lancer Fuseki en arrière-plan
cp .env.example .env
# Utilisez la commande suivante :
docker compose up -d
# Sinon, si vous voulez builder l'image vous-même, utilisez la commande suivante :
docker compose up --build
# Vérifier que le conteneur est actif
docker compose ps
# Voir les logs
docker compose logs -f
# Ou executer docker compose up sans -d pour voir les logs en direct
```

Fuseki sera disponible sur **[http://localhost:3030](http://localhost:3030)**.

Le notebook gère automatiquement le chargement RDF nécessaire à la Tâche 2 :
- base explicite : `ontology/sysdereco.owl` + `ontology/sysdereco.ttl` ;
- scénario inféré : même base + application `R1 -> R2 -> R3 -> R4 -> R5`.

Le chargement manuel via l'interface web Fuseki reste possible pour inspection/debug, mais il n'est pas requis pour l'exécution standard du notebook.

### 8. Exécution recommandée de la Tâche 2 (comparaison base vs inférence)

Le notebook implémente une section dédiée qui automatise le flux suivant :

1. **Scénario base explicite** : `sysdereco.owl + sysdereco.ttl` (sans R1-R5)
2. **Scénario avec inférence** : `sysdereco.owl + sysdereco.ttl` puis application `R1 -> R2 -> R3 -> R4 -> R5`
3. Calcul des métriques top-N du scénario KG avec le même protocole que T0/T1
4. Tableau comparatif final `T0/T1/T2`
5. Validation ciblée de `R5` (delta sur `mayLike` avec fallback minimal)

Ce flux répond à l'exigence du sujet de comparer les résultats **sans inférence** puis **avec inférence**.

Pour accélérer l'exécution, la section comparative T2 fonctionne en **single-load** :
- un chargement du dataset de base ;
- puis transition `base -> inference` sans second `CLEAR+LOAD`.

La section d'export en fin de notebook produit les artefacts suivants dans `results/` :

- `task2_kg_scenarios_summary.csv`
- `task2_t0_t1_t2_comparison.csv`
- `task2_advanced_queries_base_vs_inference.csv`
- `task2_results_bundle.json`

---

## Arrêter les services

```bash
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

- Ontologie OWL : classes `User`, `Book`, `Category`, `Rating`, `Publisher`, `Language`
- Propriétés d'objet : `ratesBook`, `isRatedBy`, `hasCategory`, `hasInteractedWith`, `mayLike`, `likesCategory`, `inLanguage`, `publishedBy`, `writtenBy`
- Propriétés de données : `name`, `title`, `subtitle`, `description`, `ratingValue`, `averageRating`, `bookId`, `userId`, `publisherName`, `price`, `languageName`
- Règles d'inférence :
  - **R1** : *Si un utilisateur a noté un livre d’une catégorie avec une note supérieure ou égale à 4.0, alors cette catégorie lui plaît.*
    ```
    User(?u) ∧ Rating(?r) ∧ Book(?i) ∧ Category(?c) ∧ isRatedBy(?r, ?u) ∧ ratesBook(?r, ?i) ∧ hasCategory(?i, ?c) ∧ ratingValue(?r, ?rating) ∧ swrlb:greaterThanOrEqual(?rating, 4.0) -> likesCategory(?u, ?c)
    ```
  - **R2** : *Si un utilisateur aime une catégorie et qu'un livre appartient à cette catégorie, alors ce livre peut être recommandé à l'utilisateur.*
    ```
    User(?u) ∧ Book(?i) ∧ Category(?c) ∧ likesCategory(?u, ?c) ∧ hasCategory(?i, ?c) -> mayLike(?u, ?i)
    ```
  - **R3** : *Si deux utilisateurs aiment la même catégorie, alors ils ont des goûts similaires.*
    ```
    User(?u1) ∧ User(?u2) ∧ Category(?c) ∧ differentFrom(?u1, ?u2) ∧ likesCategory(?u1, ?c) ∧ likesCategory(?u2, ?c) -> similarTaste(?u1, ?u2)
    ```
  - **R4** : *Si un livre d'une catégorie a une note moyenne supérieure ou égale à 4.0, alors il est considéré comme populaire dans cette catégorie.*
    ```
    Book(?i) ∧ Category(?c) ∧ hasCategory(?i, ?c) ∧ averageRating(?i, ?avgRating) ∧ swrlb:greaterThanOrEqual(?avgRating, 4.0) -> popularInCategory(?i, ?c)
    ```
  - **R5** : *Si un utilisateur A aime une catégorie et qu'un utilisateur B avec des goûts similaires a noté un livre de cette catégorie avec une note supérieure ou égale à 4.0, alors ce livre peut être recommandé à l'utilisateur A*
    ```
    User(?u1) ∧ User(?u2) ∧ Book(?i) ∧ Category(?c) ∧ Rating(?r) ∧ similarTaste(?u1, ?u2) ∧ likesCategory(?u1, ?c) ∧ isRatedBy(?r, ?u2) ∧ ratesBook(?r, ?i) ∧ hasCategory(?i, ?c) ∧ ratingValue(?r, ?rating) ∧ swrlb:greaterThanOrEqual(?rating, 4.0) -> mayLike(?u1, ?i)
    ```
- Triplestore : Apache Jena Fuseki (via Docker compose)
- Interrogation : SPARQL via Fuseki ou lignes de commande

**Exemples de requêtes SPARQL suite à l'ajout de règles d'inférence**
Lister les liaisons entre utilisateurs et catégories :
```sql
PREFIX reco: <http://www.semanticweb.org/candhigomvie/ontologies/2026/3/inf6083-sysdereco#>
SELECT ?user ?category WHERE {
  ?user reco:likesCategory ?category .
}
```


