# HealthApp-Pipeline-snowflake
# 🏥 HealthApp
HealthApp est une application Android générant des données de santé et d’activité.
Ce projet exploite plus de 10 jours de logs réels pour construire un pipeline de données avec Snowflake, transformant des données brutes en analyses exploitables.

# 🎯 Objectif du projet

L’objectif est de concevoir et implémenter un pipeline de données avec Snowflake afin de :

- Ingérer des logs bruts de l’application

- Nettoyer et préparer les données

- Transformer les données en tables structurées

- Produire des analyses et indicateurs exploitables

# 📱 Description des logs HealthApp
## 📄 Format des données

Les données utilisées dans ce projet proviennent de journaux applicatifs (logs) générés par l’application mobile HealthApp sur Android.

Chaque ligne du dataset correspond à un événement enregistré par l’application et contient les champs suivants :
| Champ           | Description                                   |
| --------------- | --------------------------------------------- |
| `LineId`        | Identifiant unique de la ligne                |
| `Time`          | Horodatage de l’événement                     |
| `Component`     | Composant de l’application à l’origine du log |
| `Pid`           | Identifiant du processus                      |
| `Content`       | Message brut du log                           |
| `EventId`       | Identifiant de l’événement                    |
| `EventTemplate` | Modèle du message (template structuré)        |

## 🔍 Types d’événements observés

Les logs contiennent plusieurs types d’informations liées à l’activité utilisateur :

🏃 Activité physique

- Nombre de pas (onStandStepChanged)

- Détail des pas journaliers (getTodayTotalDetailSteps)

🔥 Données de santé

- Calories brûlées (calculateCaloriesWithCache)

- Altitude (calculateAltitudeWithCache)

📱 Événements système

- Écran allumé (SCREEN_ON)

- Réception d’événements Android (onReceive)

⚙️ Traitements internes

- Mise à jour des données (setTodayTotalDetailSteps)

- Calculs internes de l’application

# Snowflake 
Snowflake a été choisi pour sa capacité à gérer des données semi-structurées, sa scalabilité et sa simplicité d’utilisation. Il permet de charger rapidement des logs bruts et de les transformer en données analytiques via une approche ELT moderne.

Si vous voulez reproduire ce projet, il vous faudra créer un compte [snowflake](https://signup.snowflake.com/?_l=fr) 

# Chargement des données
Dans un premier temps, afin d’organiser les scripts SQL, nous créons un dossier nommé scripts.

Ce dossier regroupe l’ensemble des scripts SQL du projet, permettant :

- d’avoir une structure claire et organisée

- de faciliter la maintenance du code

- d’adopter de bonnes pratiques professionnelles

- Chaque script correspond à une étape du pipeline de données (ingestion, transformation, modélisation).

## init
Nous commençons par créer un premier script SQL permettant de définir les éléments fondamentaux de notre environnement de travail dans Snowflake.

Ce script inclut :

- la création de la base de données

- la création du schéma

- la création de la table principale raw_event

La table raw_event est utilisée pour stocker les données brutes issues des logs de l’application HealthApp. Elle constitue le point d’entrée du pipeline de données.

<img width="312" height="343" alt="image" src="https://github.com/user-attachments/assets/c88daead-6b37-458a-94f3-6aeb48a66812" />

```
CREATE OR ALTER DATABASE HEALTH_APP_V1;

CREATE OR ALTER SCHEMA raw;

CREATE OR ALTER TABLE raw.raw_events (
    event_timestamp TIMESTAMP,
    process_name STRING,
    process_id NUMBER,
    message STRING
);
```

## 📥 Intégration des fichiers dans Snowflake

Nos fichiers de données sont stockés localement et peuvent être de formats variés, tels que CSV ou JSON.

Pour les ingérer dans Snowflake, nous avons choisi d’utiliser un internal stage.

Un stage est un espace de stockage temporaire dans Snowflake, qui permet de :

- centraliser les fichiers à ingérer

- automatiser facilement les processus de chargement ultérieurs

- séparer les fichiers bruts du reste du pipeline

Cette approche facilite l’intégration initiale des données et prépare le terrain pour des actualisations automatiques à l’avenir.

Nous allons par donc créer un script SQL 'internal stage.sql' et y mettre notre script.

```
CREATE OR ALTER STAGE raw.internal_stage
FILE_FORMAT = raw.csv_format;
```

Comme vous pouvez le constater, dans la définition du FILE_FORMAT, la valeur fait référence à un autre script SQL situé dans le schéma RAW.

J’ai choisi cette approche pour plusieurs raisons :

- Réduction de la redondance : si nous avons plusieurs fichiers CSV à ingérer, nous n’avons pas besoin de réécrire à chaque fois le type de fichier, le séparateur ou le format de timestamp.

- Centralisation de la configuration : toutes les spécifications du format de fichier sont regroupées dans un seul objet SQL (FILE_FORMAT).

- Maintenance facilitée : toute modification du format de fichier (par exemple, changement du séparateur ou du timestamp) se fait une seule fois dans le script FILE_FORMAT, et elle s’applique à tous les fichiers utilisant ce format.

Cette approche est conforme aux bonnes pratiques de Snowflake et permet de rendre le pipeline plus modulaire, réutilisable et facile à maintenir

--> Création du script file format.sql dans notre dossier principal
```
CREATE OR ALTER FILE FORMAT raw.csv_format
TYPE=CSV
FIELD_DElIMITER='|'
TIMESTAMP_FORMAT='YYYYMMDD-HH24:MI:SS:FF3';
```
Maintenant via snowflake, nous allons mettre notre 1er fichier csv dans notre stage
<img width="2310" height="1111" alt="image" src="https://github.com/user-attachments/assets/ed86201b-8962-4eef-a79d-b0236fa80f6f" />

et grâce à ```List @raw.internal_stage;```, nous pouvons observer la présence de notre nouveau fichier.

<img width="1968" height="147" alt="image" src="https://github.com/user-attachments/assets/23903271-0967-4c07-bc11-4648b85d4c8f" />

## 📥 Chargement des données dans Snowflake
Pour ingérer les fichiers CSV stockés dans notre internal stage, nous créons un script intitulé insertion_data_via_stage.sql.

Ce script utilise la commande COPY INTO de Snowflake pour transférer les données depuis le stage vers notre table principale raw.raw_events.
```
COPY INTO raw.raw_events (event_timestamp, process_name, process_id, message)
FROM (
    SELECT 
         $1 as event_timestamp,
         $2 as process_name,
         $3 as process_id,
         $4 as message
    FROM
        @raw.internal_stage
);

SELECT COUNT(*) FROM raw.raw_events;
```
<img width="1966" height="140" alt="image" src="https://github.com/user-attachments/assets/f7993318-5ad8-49da-8e88-e13fbe680c33" />

## 🔄Automatisation du chargement avec Snowpipe
Maintenant que notre internal stage est configuré et que la commande COPY INTO fonctionne correctement, l’objectif est d’automatiser le chargement des données afin d’éviter toute intervention manuelle.

Pour cela, nous utilisons Snowpipe, une fonctionnalité de Snowflake permettant l’ingestion continue des données.


⚙️ Pourquoi utiliser Snowpipe ?

Jusqu’à présent, le chargement des données nécessitait l’exécution manuelle de la commande COPY INTO.

Avec Snowpipe, ce processus devient automatique :

- Les nouveaux fichiers déposés dans le stage sont détectés automatiquement

- Les données sont chargées en continu dans la table cible

- Le pipeline devient temps réel ou quasi temps réel
  

🚀 Fonctionnement

Snowpipe repose sur le principe suivant :

- Un fichier est déposé dans le stage (internal_stage)

- Snowpipe détecte ce nouveau fichier

- Le COPY INTO est exécuté automatiquement

- Les données sont insérées dans la table raw.raw_events

=> Nous allons créer un nouveau script SQL `snowpipe.sql`
```
CREATE PIPE raw.load_raw_data
    AUTO_INGEST = TRUE
    AS
        COPY INTO raw.raw_events (event_timestamp, process_name, process_id, message)
        FROM (
            SELECT
                $1 AS event_timestamp,
                $2 AS process_name,
                $3 AS process_id,
                $4 AS message 
            FROM
                @raw.internal_stage
        )
    FILE_FORMAT = raw.csv_format;
```

Nous allons y mettre notre fichier avec un nouveau nom pour voir si le pipe fonctionne.

<img width="1482" height="463" alt="image" src="https://github.com/user-attachments/assets/e2f27fda-902c-4bfe-bc79-c7462faa7d6f" />
