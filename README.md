# INF6083 – Projet P3 : Filtrages collaboratif et basé sur la connaissance

**Cours** : INF6083 – Systèmes de recommandation · Hiver 2026  
**Date de remise** : 5 avril 2026

---

## Structure du projet

```
.
├── notebook_devoir3.ipynb         # Notebook principal
├── Dockerfile
├── docker compose.yml             # Apache Fuseki (triplestore RDF)
├── requirements.txt
├── .env                           # Variables d'environnement (URL Fuseki, etc.)
├── data/
│   └── joining/                   
│       └── temporal_pre_split     # Données P2
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

Fuseki sera disponible sur **[http://localhost:3030](http://localhost:3030)**

Interface web - Chargement des ontologies et triplets
1. Créer un dataset `recommendations` (type persistant `TDB2`)
2. Dans la gestion des datasets, accéder à l'ajout de données `add data`
3. Charger les deux fichiers :
    - Étape 1 - Ontologie de base : `ontology/sysdereco.owl`& `ontology/sysdereco.ttl`
    - Étape 2 - Ontologie avec règles d'inférence : `ontology/sysdereco_inferred.owl` & `ontology/sysdereco.ttl`
4. Vérifier dans `query` le bon chargement des fichiers (nombre de triplets > 0) :
```sql
# Nombre total de triplets
SELECT (COUNT(*) as ?count) {?s ?p ?o}
```
Exemple de résultat (type Response) :
```bash
{
  "head": {
    "vars": [
      "count"
    ]
  },
  "results": {
    "bindings": [
      {
        "count": {
          "type": "literal",
          "datatype": "http://www.w3.org/2001/XMLSchema#integer",
          "value": "1400582"
        }
      }
    ]
  }
}
```
Pour l'étape des règles d'inférence, ces dernières devront être appliquées depuis le notebook (R1 -> R2 -> R3 -> R4 -> R5). L'ordre est important car les dernières règles dépendent des résultats obtenus des précédentes.

---

## Arrêter les services

```bash
# Arrêter Fuseki
docker-compose down

# Arrêter et supprimer les données persistantes
docker-compose down -v
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


