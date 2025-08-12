# Pipeline de Traitement BigQuery avec Dataflow et Google Pub/Sub

## Introduction

Cette solution est conçue pour gérer des transactions à travers un pipeline Dataflow. Le pipeline extrait des données depuis plusieurs tables BigQuery, principalement des tables de transactions, puis les traite et les publie via Pub/Sub. Les informations transformées sont ensuite intégrées dans une table de consolidation. Ce système repose sur Flask pour l’interface d’API, Apache Beam pour le traitement des données, et utilise les services Google Cloud pour le stockage, la publication de messages et la gestion des requêtes programmées.

## Structure de la Solution

1. **Cloud Scheduler** :
    - Programme et déclenche automatiquement des requêtes HTTP toutes les 20 minutes pour exécuter le pipeline via l’API Flask.
2. **API Flask** :
    - Interface REST permettant de déclencher le pipeline via une requête HTTP.
3. **Google BigQuery** :
    - Stockage des données des Transactions.
4. **Google Cloud Pub/Sub** :
    - Pub/Sub est utilisé pour transmettre les IDs des opérations à traiter.
5. **Apache Beam & Dataflow** :
    - Traitement de données à grande échelle sur Dataflow pour filtrer, transformer, et insérer les données consolidées.
6. **Google Cloud Storage** :
    - Stocke un fichier de timestamp utilisé pour synchroniser les extractions.

![Architecture](./images/architecture.png)




## Guide d'Installation et de Configuration

### Prérequis

- **Google Cloud SDK** installé et configuré.
- **Service Account** avec les rôles suivants : BigQuery Admin, Pub/Sub Editor, Storage Admin, et Dataflow Developer.
- **Droits d’accès** sur le projet GCP.

### Configuration des Services

- **Cloud Scheduler** :
    - Configurer un job qui envoie une requête HTTP vers l’API `/run_pipeline` toutes les x minutes.
- **BigQuery** :
    - Avoir les tables de source (`ussd_operations`, `transactiongu`, etc.) et la table de destination `consolidated_table` dans un dataset (`consolidated_dataset`).
- **Pub/Sub** :
    - Créer un topic `ussd-operations-topic`.
    - Configurer une subscription `ussd-operations-topic-sub` pour recevoir les messages du topic.
- **Cloud Storage** :
    - Créer un bucket (`infra-intouch`) pour les fichiers temporaires (ex. `last_timestamp.txt`).
- **Dataflow** :
    - S’assurer que Dataflow est activé dans le projet GCP.
    - Configurer un bucket pour la staging et la temp location.

## Description de l’Architecture

1. **Déclenchement de l’API** :
    - L'API `/run_pipeline` permet de déclencher manuellement le pipeline avec une requête HTTP.
    - Une réponse est envoyée  pour confirmer le lancement du pipeline.
2. **Extraction des Opérations (BigQuery)** :
    - Une requête SQL extrait les données depuis BigQuery où les `source_timestamp` sont plus récents que le dernier timestamp sauvegardé.
3. **Publication via Pub/Sub** :
    - Chaque opération est publiée comme message dans Pub/Sub, contenant l'ID de l'opération.
    - Ce mécanisme permet de traiter chaque opération de manière indépendante en parallèle.
4. **Extraction des IDs depuis Pub/Sub** :
    - Les messages sont récupérés via une subscription, et leurs IDs sont extraits pour filtrer les opérations à traiter.
5. **Traitement Apache Beam & Dataflow** :
    - Utilisation d’Apache Beam pour lire, transformer et écrire les données dans BigQuery.
    - Filtrage des opérations selon les IDs publiés, enrichissement avec des jointures sur d’autres tables.
6. **Consolidation dans BigQuery** :
    - Les données consolidées sont écrites dans une table temporaire (`temp_table`).
    - Une opération `MERGE` intègre ensuite les données de la `temp_table` vers la `consolidated_table`.

## Explication du Code

### 1. `publish_new_ussd_operations()`

Cette fonction extrait les opérations nouvelles depuis `ussd_operations` dans BigQuery en comparant les timestamps. Les opérations extraites sont publiées dans Pub/Sub. Le fichier `last_timestamp.txt` est mis à jour avec le timestamp le plus récent traité.

### Structure

- **Entrée** : `last_processed_timestamp` (via Cloud Storage).
- **Sortie** : Publie chaque opération dans `ussd-operations-topic`.

### 2. `get_ids_from_pubsub()`

Cette fonction récupère les messages depuis le topic Pub/Sub `ussd-operations-topic` et en extrait les IDs pour les utiliser comme filtres dans la requête BigQuery du pipeline Dataflow.

### Structure

- **Entrée** : Subscription à `ussd-operations-topic-sub`.
- **Sortie** : Liste `ids` des IDs extraits.

### 3. `merge_consolidated_tables()`

Cette fonction effectue un `MERGE` entre une `temp_table` et la `consolidated_table`. Elle vérifie si `temp_table` contient des données et applique l’insertion ou la mise à jour dans `consolidated_table`.

### Structure

- **Entrée** : Requête SQL pour la vérification de `temp_table`.
- **Sortie** : Consolidation des données dans `consolidated_table`.

### 4. `run_pipeline()`

Le point d’entrée principal qui orchestre les étapes précédentes et exécut
