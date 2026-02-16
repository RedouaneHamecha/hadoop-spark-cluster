![Cluster Big Data Local](bg.jpg)

# Cluster Big Data Local — Hadoop + Spark + Jupyter

## Quel est l'objet de ce projet ?

Ce projet simule sur **une seule machine personnelle** ce qu'une grande entreprise ferait avec des dizaines de serveurs. Il permet de traiter de grandes quantités de données de façon organisée, performante et reproductible, en utilisant les mêmes technologies que Netflix, Airbnb ou Spotify utilisent en production.

Concrètement, on a mis en place un environnement complet qui permet d'écrire du code, de stocker des données volumineuses, et de les traiter en parallèle — le tout sans dépendre d'internet, sans payer de service cloud, et sans installer quoi que ce soit directement sur la machine.

---

## Comment on a simulé plusieurs serveurs sur une seule machine ?

C'est la question centrale. La réponse tient en un mot : **Docker**.

Docker est un logiciel qui permet de créer des **conteneurs**. Un conteneur c'est une sorte de compartiment étanche qui tourne à l'intérieur de l'ordinateur. Chaque conteneur se comporte comme une machine indépendante avec son propre système, ses propres programmes, sa propre mémoire allouée. Plusieurs conteneurs peuvent tourner en même temps sur le même ordinateur, chacun ignorant l'existence des autres, sauf si on leur dit explicitement de communiquer.

C'est ainsi qu'on a simulé 8 serveurs sur une seule machine :

```
Un seul ordinateur physique
│
├── Conteneur 1  →  se comporte comme un serveur de gestion HDFS    (namenode)
├── Conteneur 2  →  se comporte comme un serveur de stockage        (datanode1)
├── Conteneur 3  →  se comporte comme un second serveur de stockage (datanode2)
├── Conteneur 4  →  se comporte comme un serveur de coordination    (spark-master)
├── Conteneur 5  →  se comporte comme un serveur de calcul          (spark-worker)
├── Conteneur 6  →  se comporte comme un serveur de calcul          (spark-worker-2)
├── Conteneur 7  →  se comporte comme un serveur de calcul          (spark-worker-3)
└── Conteneur 8  →  se comporte comme un serveur d'interface        (jupyter)
```

Ces 8 conteneurs se parlent via un réseau interne créé par Docker, exactement comme des vrais serveurs se parleraient via un réseau d'entreprise.

---

## Pourquoi pas simplement Google Colab ou Jupyter sur son PC ?

### Google Colab
Google Colab est un outil d'analyse de données qui tourne sur les serveurs de Google, accessible depuis le navigateur.

Ses limites sont précises :

- **Les données disparaissent** à chaque session.

- **La mémoire est limitée** à environ 12 Go.

- **Internet est obligatoire**. 

- **Aucun contrôle sur les ressources**. Google décide de la puissance disponible selon les forfait payants

- **Les données transitent par les serveurs de Google**. problèmes de confidentialité

### Jupyter directement sur son PC
Jupyter installé localement règle le problème de confidentialité, mais reste limité par la machine sur laquelle il tourne.

- Tout le traitement repose sur **un seul processeur** et dans **la mémoire de la machine**.
- Impossible de traiter un fichier plus lourd que la mémoire disponible.
- Le travail n'est jamais distribué sur plusieurs unités — tout est séquentiel.

---

## À quel moment ce cluster devient-il pertinent ?

| Situation | Ce que ce cluster apporte |
|-----------|--------------------------|
| Fichiers de données volumineux (> 2 Go) | Le traitement est distribué sur plusieurs conteneurs en parallèle |
| Données confidentielles ou sensibles | Rien ne sort de la machine, aucune donnée ne transite par internet |
| Travail en équipe | Un seul fichier de configuration à partager — tout le monde a exactement le même environnement |
| Besoin de résultats reproductibles | L'environnement est identique à chaque démarrage, sur n'importe quelle machine |

---

**Pour des fichiers légers et des analyses simples**, Colab ou Jupyter local restent plus rapides à utiliser. Ce cluster devient pertinent dès que le volume de données ou les contraintes de confidentialité dépassent ce que ces outils peuvent offrir.

---

## Vue globale de l'infrastructure

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                          ORDINATEUR PHYSIQUE (Windows)                              ║
║                                                                                      ║
║  ┌────────────────────────────────────────────────────────────────────────────────┐  ║
║  │                         DOCKER DESKTOP                                        │  ║
║  │                                                                                │  ║
║  │  ┌─────────────────────────────────────────────────────────────────────────┐  │  ║
║  │  │                      RÉSEAU INTERNE : hadoop-network                    │  │  ║
║  │  │                                                                         │  │  ║
║  │  │   ╔═══════════════╗        ╔══════════════╗     ╔══════════════╗        │  │  ║
║  │  │   ║   NAMENODE    ║        ║  DATANODE 1  ║     ║  DATANODE 2  ║        │  │  ║
║  │  │   ║               ║◄──────►║              ║     ║              ║        │  │  ║
║  │  │   ║ port 9870     ║        ║   stockage   ║     ║   stockage   ║        │  │  ║
║  │  │   ║ port 9000     ║◄──────►║   + backup   ║◄───►║   + backup   ║        │  │  ║
║  │  │   ╚═══════════════╝        ╚══════════════╝     ╚══════════════╝        │  │  ║
║  │  │          ▲                                                               │  │  ║
║  │  │          │              COUCHE STOCKAGE — HDFS                          │  │  ║
║  │  │          │                                                               │  │  ║
║  │  │  ════════╪═══════════════════════════════════════════════════════════    │  │  ║
║  │  │          │                                                               │  │  ║
║  │  │          │              COUCHE CALCUL — SPARK                           │  │  ║
║  │  │          │                                                               │  │  ║
║  │  │          ▼                                                               │  │  ║
║  │  │   ╔═══════════════╗                                                      │  │  ║
║  │  │   ║  SPARK MASTER ║◄─────────────────────────────────────────────────┐  │  │  ║
║  │  │   ║               ║                                                   │  │  │  ║
║  │  │   ║  port 8080    ║        ╔══════════════╗                           │  │  │  ║
║  │  │   ║  port 7077    ║◄──────►║   WORKER 1   ║ port 8081                │  │  │  ║
║  │  │   ║               ║        ║  2c / 2Go RAM║                           │  │  │  ║
║  │  │   ║               ║        ╚══════════════╝                           │  │  │  ║
║  │  │   ║               ║        ╔══════════════╗                           │  │  │  ║
║  │  │   ║               ║◄──────►║   WORKER 2   ║ port 8082                │  │  │  ║
║  │  │   ║               ║        ║  2c / 2Go RAM║                           │  │  │  ║
║  │  │   ║               ║        ╚══════════════╝                           │  │  │  ║
║  │  │   ║               ║        ╔══════════════╗                           │  │  │  ║
║  │  │   ║               ║◄──────►║   WORKER 3   ║ port 8083                │  │  │  ║
║  │  │   ╚═══════════════╝        ║  2c / 2Go RAM║                           │  │  │  ║
║  │  │          ▲                 ╚══════════════╝                           │  │  │  ║
║  │  │          │                                                             │  │  │  ║
║  │  │  ════════╪═══════════════════════════════════════════════════════════  │  │  │  ║
║  │  │          │                                                             │  │  │  ║
║  │  │          │              COUCHE INTERFACE — JUPYTER                    │  │  │  ║
║  │  │          │                                                             │  │  │  ║
║  │  │          ▼                                                             │  │  │  ║
║  │  │   ╔═══════════════╗                                                   │  │  │  ║
║  │  │   ║    JUPYTER    ║───────────────────────────────────────────────────┘  │  │  ║
║  │  │   ║               ║                                                      │  │  ║
║  │  │   ║  port 8888    ║                                                      │  │  ║
║  │  │   ╚═══════════════╝                                                      │  │  ║
║  │  │                                                                          │  │  ║
║  │  └──────────────────────────────────────────────────────────────────────────┘  │  ║
║  └────────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                      ║
║   PORTS ACCESSIBLES DEPUIS TON NAVIGATEUR :                                         ║
║   localhost:9870  →  Interface web HDFS    (voir les fichiers)                       ║
║   localhost:8080  →  Interface web Spark   (voir les jobs)                           ║
║   localhost:8081  →  Interface web Worker 1                                          ║
║   localhost:8082  →  Interface web Worker 2                                          ║
║   localhost:8083  →  Interface web Worker 3                                          ║
║   localhost:8888  →  Jupyter Notebook      (écrire le code)                          ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

---

## Comment c'est organisé ?

L'environnement repose sur deux fonctions distinctes : **stocker les données** et **les traiter**.

### Le stockage — HDFS (Hadoop Distributed File System) 

Dans un vrai datacenter, le stockage distribué repose sur des racks de serveurs physiques dédiés exclusivement à cette fonction. Ici, on simule ce même principe avec 3 conteneurs Docker.

Quand on uploade un fichier sur HDFS, il est automatiquement découpé en blocs numérotés et répartis sur les datanodes. Le namenode ne stocke aucune donnée — il garde uniquement la carte qui dit où se trouve chaque bloc.

```
                    ┌─────────────────────────────────┐
                    │            NAMENODE             │
                    │         (le cartographe)        │
                    │                                 │
                    │  fichier.csv                    │
                    │  ├── bloc 1  → datanode1        │
                    │  ├── bloc 2  → datanode2        │
                    │  ├── bloc 3  → datanode1        │
                    │  └── bloc 4  → datanode2        │
                    │                                 │
                    │  (copie de secours)             │
                    │  ├── bloc 1  → datanode2 aussi  │
                    │  ├── bloc 2  → datanode1 aussi  │
                    │  ├── bloc 3  → datanode2 aussi  │
                    │  └── bloc 4  → datanode1 aussi  │
                    └────────────┬────────────────────┘
                                 │ "où sont les blocs ?"
                  ┌──────────────┴──────────────┐
                  ↓                             ↓
    ┌─────────────────────────┐   ┌─────────────────────────┐
    │        DATANODE 1       │   │        DATANODE 2       │
    │    (serveur stockage)   │   │    (serveur stockage)   │
    │                         │   │                         │
    │  [bloc 1] [bloc 3]      │   │  [bloc 2] [bloc 4]      │
    │  [bloc 2] [bloc 4]      │   │  [bloc 1] [bloc 3]      │
    │   ↑ copies de secours   │   │   ↑ copies de secours   │
    └─────────────────────────┘   └─────────────────────────┘
```

Si datanode1 tombe en panne, toutes les données existent encore sur datanode2. Le namenode le détecte et redirige automatiquement toutes les lectures vers datanode2.

---

### Le calcul — Spark

Dans un vrai datacenter, le calcul distribué repose sur des racks de serveurs physiques dédiés au traitement. Ici, on simule ce principe avec 4 conteneurs Docker (1 master + 3 workers).

Spark découpe le travail en tâches et les envoie sur plusieurs serveurs de calcul qui travaillent simultanément. Plus il y a de serveurs de calcul, plus le traitement est rapide.

```
Calcul Spark
│
├── spark-master    → reçoit les demandes et distribue les tâches
├── spark-worker    → exécute les calculs (2 cœurs, 2 Go RAM)
├── spark-worker-2  → exécute les calculs (2 cœurs, 2 Go RAM)
└── spark-worker-3  → exécute les calculs (2 cœurs, 2 Go RAM)
```

---

### L'interface — Jupyter

Jupyter est la page web depuis laquelle on écrit et exécute le code. Jupyter ne fait aucun calcul lui-même — il transmet les instructions à Spark, qui va chercher les données dans HDFS, traite tout, et renvoie le résultat.

---

### Trois langages dans le même notebook

Jupyter supporte Python, PySpark et SQL dans le même notebook — chaque cellule peut utiliser un langage différent selon le besoin.

| Langage | Usage | Exemple |
|---------|-------|---------|
| Python | visualisation, petits fichiers locaux | `df.plot()` |
| PySpark | traitement distribué sur le cluster | `df.groupBy("col").count().show()` |
| SQL | requêtes sur les données Spark | `spark.sql("SELECT * FROM table").show()` |

Charger (PySpark) → Agréger (SQL) → Visualiser (Python)

---

## Le trajet d'une instruction, du début à la fin

```
On écrit une instruction dans Jupyter
               ↓
┌──────────────────────────────────────┐
│             JUPYTER                  │
│       localhost:8888                 │
│  "Lis ce fichier et calcule la       │
│   moyenne de cette colonne"          │
└──────────────────────────────────────┘
               ↓
┌──────────────────────────────────────┐
│          SPARK MASTER                │
│  "Je découpe le travail en 3 tâches  │
│   et je les envoie aux 3 workers"    │
└──────────────────────────────────────┘
        ↓             ↓            ↓
┌──────────┐   ┌──────────┐   ┌──────────┐
│ WORKER 1 │   │ WORKER 2 │   │ WORKER 3 │
│ tâche 1  │   │ tâche 2  │   │ tâche 3  │
└──────────┘   └──────────┘   └──────────┘
        ↓             ↓            ↓
┌──────────────────────────────────────┐
│               HDFS                   │
│  namenode indique où sont les blocs  │
│  datanodes envoient les données      │
│  à chaque worker qui en a besoin     │
└──────────────────────────────────────┘
        ↓             ↓            ↓
┌──────────┐   ┌──────────┐   ┌──────────┐
│ WORKER 1 │   │ WORKER 2 │   │ WORKER 3 │
│ calcule  │   │ calcule  │   │ calcule  │
│ sa part  │   │ sa part  │   │ sa part  │
└──────────┘   └──────────┘   └──────────┘
        ↓             ↓            ↓
┌──────────────────────────────────────┐
│          SPARK MASTER                │
│  "Je rassemble les 3 résultats"      │
└──────────────────────────────────────┘
               ↓
    Résultat affiché dans Jupyter
```

---

## Les limites de cette simulation

Simuler plusieurs serveurs sur une seule machine a des limites qu'il faut connaître.

**Les ressources sont partagées.** Dans un vrai cluster, chaque serveur a sa propre alimentation, son propre processeur, sa propre mémoire. Ici, tous les conteneurs se partagent les ressources d'un seul ordinateur. Si la machine a 16 Go de RAM et que les conteneurs en demandent 14, la machine devient lente.

**La panne n'est pas vraiment isolée.** Si l'ordinateur physique s'éteint, tout s'éteint en même temps — namenode, datanodes, spark, tout. Dans un vrai cluster, les serveurs sont indépendants et une panne sur l'un n'affecte pas les autres.

**La puissance réseau est simulée.** Dans un vrai datacenter, les serveurs communiquent via des câbles réseau à très haute vitesse. Ici, la communication entre conteneurs passe par le réseau interne de Docker, qui est plus lent qu'une vraie infrastructure réseau physique.

**Le volume de données reste limité** par la capacité de stockage et la mémoire de la machine hôte.

---

## La scalabilité — jusqu'où peut-on aller ?

L'un des avantages de cette architecture est qu'elle est extensible sans tout reconfigurer.

Ajouter de la puissance de calcul revient à ajouter un nouveau conteneur `spark-worker` dans le fichier de configuration. Ajouter de la capacité de stockage revient à ajouter un nouveau conteneur `datanode`. Pour chaque machine ajoutée, on définit précisément la quantité de **RAM** et le **nombre de cœurs CPU** qu'elle est autorisée à utiliser — selon ce que la machine physique peut réellement fournir.

Cette même logique s'applique à un vrai cluster cloud : on ajoute des serveurs physiques ou virtuels à la demande, avec les ressources souhaitées, sans toucher à l'architecture existante.

---

## Les bénéfices

**Confidentialité totale.** Aucune donnée ne quitte la machine. Aucun compte, aucun abonnement, aucun serveur tiers impliqué.

**Environnement identique partout.** Un seul fichier de configuration suffit pour que n'importe qui, sur n'importe quelle machine, obtienne exactement le même environnement. Fini les problèmes de compatibilité entre machines.

**Infrastructure réelle, coût zéro.** On utilise les mêmes technologies que celles déployées dans les grandes infrastructures de données, sans aucun coût matériel ni d'abonnement.

**Résilience des données.** Les données et les fichiers de travail sont sauvegardés sur l'ordinateur physique. Supprimer ou reconstruire les conteneurs ne fait perdre aucune donnée.

---

*Pour le guide de démarrage et d'utilisation, voir le fichier `GUIDE-DEMARRAGE.txt`*
*Pour les détails techniques de configuration et l'annuaire PySpark, voir le fichier `README-TECHNIQUE.txt`*

---

# Sources des images Docker

| Image | Auteur | Lien |
|-------|--------|------|
| `bde2020/hadoop-namenode` | Big Data Europe | https://hub.docker.com/r/bde2020/hadoop-namenode |
| `bde2020/hadoop-datanode` | Big Data Europe | https://hub.docker.com/r/bde2020/hadoop-datanode |
| `bde2020/spark-master` | Big Data Europe | https://hub.docker.com/r/bde2020/spark-master |
| `bde2020/spark-worker` | Big Data Europe | https://hub.docker.com/r/bde2020/spark-worker |
| `jupyter/pyspark-notebook` | Project Jupyter | https://hub.docker.com/r/jupyter/pyspark-notebook |

Les images `bde2020` sont maintenues par le projet [Big Data Europe](https://github.com/big-data-europe/docker-hadoop) et distribuées sous licence Apache 2.0.
L'image `jupyter/pyspark-notebook` est maintenue par le projet [Jupyter Docker Stacks](https://github.com/jupyter/docker-stacks) et distribuée sous licence BSD 3-Clause.
