==============================================================================
  README TECHNIQUE — Hadoop 3.2 + Spark 3.1.2 + Jupyter
  Configuration, modification et référence PySpark
==============================================================================


------------------------------------------------------------------------------
  STRUCTURE DU PROJET
------------------------------------------------------------------------------

      hadoop-spark-cluster/
      │
      ├── docker-compose.yml     définit et configure tous les conteneurs
      ├── hadoop.env             variables de configuration Hadoop
      ├── data/                  dossier partagé entre tous les conteneurs et le PC
      ├── notebooks/             dossier des notebooks Jupyter (sauvegardé sur le PC)
      └── scripts/               dossier pour les scripts Spark


------------------------------------------------------------------------------
  COMPRENDRE LA STRUCTURE D'UN CONTENEUR DANS docker-compose.yml
------------------------------------------------------------------------------

  Chaque conteneur du cluster est décrit par un bloc dans ce fichier.
  Voici comment lire et modifier chaque propriété :

      nom-du-service:
        image: image:version
        #  ↑ programme à télécharger + sa version
        #    NE PAS mélanger les versions Spark entre conteneurs

        container_name: nom
        #  ↑ nom visible dans Docker Desktop

        hostname: nom
        #  ↑ nom que les AUTRES conteneurs utilisent pour le trouver sur le réseau

        restart: always
        #  ↑ always = redémarre seul si crash | no = ne redémarre pas

        ports:
          - "PORT_PC:PORT_CONTENEUR"
          #   ↑           ↑
          #   port que tu  port fixe à l'intérieur du conteneur
          #   tapes dans   NE JAMAIS changer le côté droit
          #   le navigateur

        environment:
          - VARIABLE=valeur
          #  ↑ paramètres passés au programme au démarrage

        volumes:
          - ./dossier_local:/dossier_conteneur
          #   ↑               ↑
          #   chemin sur ton PC  chemin vu depuis le conteneur
          #   (relatif au fichier docker-compose.yml)

        networks:
          - hadoop-network
          #  ↑ tous les conteneurs avec ce réseau peuvent se parler

        depends_on:
          - autre-service
          #  ↑ Docker attend que cet autre conteneur soit prêt avant de démarrer


------------------------------------------------------------------------------
  CE QU'ON PEUT CHANGER ET COMMENT
------------------------------------------------------------------------------


  CHANGER LE MOT DE PASSE JUPYTER
  ─────────────────────────────────
  Dans le bloc jupyter de docker-compose.yml, modifier JUPYTER_TOKEN :

      jupyter:
        environment:
          - JUPYTER_TOKEN=bonjour    ← changer "bonjour" par le mot de passe voulu

  Après modification, relancer le cluster :

      docker-compose down
      docker-compose up -d


  CHANGER UN PORT ACCESSIBLE DEPUIS LE NAVIGATEUR
  ─────────────────────────────────────────────────
  Si un port est déjà utilisé par un autre programme sur le PC,
  changer UNIQUEMENT le côté gauche :

      ports:
        - "9871:9870"    ← si le port 9870 est déjà pris sur le PC
        - "9001:9000"    ← si le port 9000 est déjà pris sur le PC
      #    ↑    ↑
      #    PC   conteneur (ne jamais toucher)

  Le navigateur utilisera alors le nouveau port : http://localhost:9871


  CHANGER LA MÉMOIRE RAM ET LES COEURS D'UN WORKER
  ──────────────────────────────────────────────────
  Dans chaque bloc spark-worker, modifier ces deux variables :

      spark-worker:
        environment:
          - SPARK_WORKER_MEMORY=2g   ← 2g = 2 Go | 4g = 4 Go | 512m = 512 Mo
          - SPARK_WORKER_CORES=2     ← nombre de tâches parallèles autorisées

  Règle : la somme de la mémoire de tous les workers ne doit pas dépasser
  la RAM disponible sur le PC, en gardant au moins 4 Go pour Windows et Docker.

      Exemple pour un PC avec 16 Go de RAM :
      16 - 4 (Windows) - 2 (Docker) = 10 Go disponibles
      3 workers × 3g = 9 Go  → acceptable
      3 workers × 4g = 12 Go → trop, le PC va ramer


  CHANGER LE DOSSIER DE SAUVEGARDE DES NOTEBOOKS
  ─────────────────────────────────────────────────
  Dans le bloc jupyter, modifier le volume :

      jupyter:
        volumes:
          - ./notebooks:/home/jovyan/work    ← remplacer "./notebooks"
          - ./data:/home/jovyan/data


  CHANGER LE DOSSIER DE DONNÉES PARTAGÉ
  ────────────────────────────────────────
  Changer "./data" dans TOUS les conteneurs qui l'utilisent :

      namenode:
        volumes:
          - ./mes-donnees:/data           ← même nouveau nom partout

      spark-master:
        volumes:
          - ./mes-donnees:/data           ← idem

      jupyter:
        volumes:
          - ./mes-donnees:/home/jovyan/data   ← idem


  CHANGER LE NOM DU CLUSTER
  ──────────────────────────
      namenode:
        environment:
          - CLUSTER_NAME=hadoop-cluster   ← changer par n'importe quel nom sans espaces


------------------------------------------------------------------------------
  LA RÈGLE ABSOLUE DES VERSIONS SPARK
------------------------------------------------------------------------------

  Toutes les images Spark dans le fichier doivent avoir exactement la même
  version. Une différence même mineure provoque une erreur
  InvalidClassException — les conteneurs ne peuvent plus se comprendre.

  CORRECT — toutes identiques :
      spark-master:   image: bde2020/spark-master:3.1.2-hadoop3.2
      spark-worker:   image: bde2020/spark-worker:3.1.2-hadoop3.2
      spark-worker-2: image: bde2020/spark-worker:3.1.2-hadoop3.2
      spark-worker-3: image: bde2020/spark-worker:3.1.2-hadoop3.2
      jupyter:        image: jupyter/pyspark-notebook:spark-3.1.2

  INCORRECT — une seule différence → tout plante :
      spark-master:   image: bde2020/spark-master:3.1.1-hadoop3.2
      jupyter:        image: jupyter/pyspark-notebook:spark-3.1.2

  Si tu veux changer la version Spark, la changer partout en même temps.


------------------------------------------------------------------------------
  LA RÈGLE DES PORTS
------------------------------------------------------------------------------

  Sur le PC, deux conteneurs ne peuvent pas utiliser le même port côté gauche.
  Le côté droit (port interne du conteneur) ne change jamais.

      Conteneur       Ports             Accessible à
      ──────────────────────────────────────────────────────────────
      namenode        9870:9870         http://localhost:9870
      namenode        9000:9000         code Python (hdfs://namenode:9000)
      spark-master    8080:8080         http://localhost:8080
      spark-master    7077:7077         communication interne Spark
      spark-worker    8081:8081         http://localhost:8081
      spark-worker-2  8082:8081         http://localhost:8082
      spark-worker-3  8083:8081         http://localhost:8083
      jupyter         8888:8888         http://localhost:8888


------------------------------------------------------------------------------
  hadoop.env — CE QU'ON PEUT CHANGER
------------------------------------------------------------------------------

  Ce fichier configure le comportement de HDFS. Il est chargé par namenode,
  datanode1 et datanode2. Chaque modification nécessite un redémarrage complet.


  CHANGER LE FACTEUR DE RÉPLICATION
  ────────────────────────────────────
      HDFS_CONF_dfs_replication=2
      #  ↑ nombre de copies de chaque bloc sur les datanodes
      #  1 = pas de copie de secours (déconseillé)
      #  2 = une copie de secours (config actuelle)
      #  3 = deux copies de secours (nécessite au moins 3 datanodes)
      #
      #  Règle : la valeur ne peut pas dépasser le nombre de datanodes
      #  Avec 2 datanodes → maximum 2
      #  Avec 3 datanodes → maximum 3


  ACTIVER LES PERMISSIONS EN PRODUCTION
  ────────────────────────────────────────
      HDFS_CONF_dfs_permissions_enabled=false
      #  false = tout le monde peut lire et écrire (dev, test)
      #  true  = les droits d'accès sont vérifiés (production)


  CHANGER L'ADRESSE DU NAMENODE
  ────────────────────────────────
      CORE_CONF_fs_defaultFS=hdfs://namenode:9000
      #                              ↑        ↑
      #                    hostname du        port HDFS
      #                    conteneur namenode
      #
      #  Ne changer que si tu as renommé le conteneur namenode.
      #  Si tu le changes ici, le changer aussi dans TOUS les autres
      #  conteneurs du docker-compose.yml (CORE_CONF_fs_defaultFS).


------------------------------------------------------------------------------
  VERSIONS DES IMAGES UTILISÉES
------------------------------------------------------------------------------

      Conteneur          Image                                        Raison
      ─────────────────────────────────────────────────────────────────────────
      namenode           bde2020/hadoop-namenode:                     Dernière
                         2.0.0-hadoop3.2.1-java8                      version dispo
      ─────────────────────────────────────────────────────────────────────────
      datanode1          bde2020/hadoop-datanode:                     Même image,
      datanode2          2.0.0-hadoop3.2.1-java8                      lancée 2 fois
      ─────────────────────────────────────────────────────────────────────────
      spark-master       bde2020/spark-master:3.1.2-hadoop3.2         Compatible
                                                                       Hadoop 3.2
      ─────────────────────────────────────────────────────────────────────────
      spark-worker x3    bde2020/spark-worker:3.1.2-hadoop3.2         Identique
                                                                       au master
      ─────────────────────────────────────────────────────────────────────────
      jupyter            jupyter/pyspark-notebook:spark-3.1.2         Correspond
                                                                       à Spark 3.1.2
      ─────────────────────────────────────────────────────────────────────────

  Note : les images bde2020 Hadoop n'ont pas été mises à jour depuis 2019.
  2.0.0-hadoop3.2.1-java8 est la dernière disponible.
  Les images Spark bde2020 existent en 3.3.0 mais pour Hadoop 3.3,
  incompatible avec nos images Hadoop 3.2.


==============================================================================
  ANNUAIRE — GUIDE DE RÉFÉRENCE PYSPARK
==============================================================================

  LA RÈGLE FONDAMENTALE

      Spark traite.
      Matplotlib affiche.
      toPandas() fait le lien entre les deux.


------------------------------------------------------------------------------
  LES 3 FAÇONS D'ÉCRIRE DU CODE DANS CE CLUSTER
------------------------------------------------------------------------------

  FAÇON 1 — PySpark natif
  ─────────────────────────
  Le langage propre à Spark. Le plus performant.

      df.groupBy("col").mean().show()


  FAÇON 2 — pyspark.pandas
  ──────────────────────────
  Syntaxe Pandas exacte, exécutée par Spark en arrière-plan.
  Changer "import pandas as pd" par "import pyspark.pandas as ps"
  et tout le reste du code reste identique.

      import pyspark.pandas as ps
      df.groupby("col").mean()


  FAÇON 3 — SQL pur
  ──────────────────
  Enregistrer le dataframe comme table, puis écrire du SQL classique.

      df.createOrReplaceTempView("ma_table")
      spark.sql("SELECT col, AVG(col2) FROM ma_table GROUP BY col").show()


------------------------------------------------------------------------------
  DÉMARRAGE — À ÉCRIRE UNE SEULE FOIS AU DÉBUT DU NOTEBOOK
------------------------------------------------------------------------------

      from pyspark.sql import SparkSession

      spark = SparkSession.builder \
          .appName("MonNotebook") \
          .master("spark://spark-master:7077") \
          .getOrCreate()

  getOrCreate() récupère la session si elle existe déjà, sinon en crée une.
  Ne jamais appeler cette cellule deux fois dans le même notebook.


------------------------------------------------------------------------------
  LIRE DES DONNÉES DEPUIS HDFS
------------------------------------------------------------------------------

      # CSV
      df = spark.read.csv("hdfs://namenode:9000/data/fichier.csv",
                          header=True,       # True si le fichier a un en-tête
                          inferSchema=True)  # Spark devine le type des colonnes

      # JSON
      df = spark.read.json("hdfs://namenode:9000/data/fichier.json")

      # Parquet — format recommandé pour les gros volumes
      df = spark.read.parquet("hdfs://namenode:9000/data/dossier/")


------------------------------------------------------------------------------
  ANNUAIRE COMPLET — PYSPARK VS PANDAS
------------------------------------------------------------------------------


  LECTURE DES DONNÉES
  ─────────────────────────────────────────────────────────────────────────
  Pandas                          PySpark                    Note
  ─────────────────────────────────────────────────────────────────────────
  pd.read_csv("fichier.csv")      spark.read.csv("hdfs://")  Chemin HDFS
  pd.read_json("fichier.json")    spark.read.json("hdfs://")
  pd.read_parquet("fichier")      spark.read.parquet("hdfs://")
  ─────────────────────────────────────────────────────────────────────────


  EXPLORER LES DONNÉES
  ─────────────────────────────────────────────────────────────────────────
  Pandas                  PySpark                      Note
  ─────────────────────────────────────────────────────────────────────────
  df.head(10)             df.show(10)
  df.shape                df.count() + len(df.columns)  Deux commandes
  df.dtypes               df.printSchema()              Structure imbriquée
  df.describe()           df.describe().show()
  df.columns              df.columns                    Identique
  df.info()               Pas d'équivalent direct
  ─────────────────────────────────────────────────────────────────────────


  NETTOYER LES DONNÉES
  ─────────────────────────────────────────────────────────────────────────
  Pandas                          PySpark                    Note
  ─────────────────────────────────────────────────────────────────────────
  df.dropna()                     df.dropna()                Identique
  df.fillna(0)                    df.fillna(0)               Identique
  df.drop("col")                  df.drop("col")             Identique
  df.drop_duplicates()            df.dropDuplicates()        Majuscule au D
  df.rename(columns={"a":"b"})    df.withColumnRenamed("a","b")
  ─────────────────────────────────────────────────────────────────────────


  SÉLECTIONNER ET FILTRER
  ─────────────────────────────────────────────────────────────────────────
  Pandas                  PySpark                      Note
  ─────────────────────────────────────────────────────────────────────────
  df["colonne"]           df.select("colonne")
  df[["col1","col2"]]     df.select("col1","col2")
  df[df["col"] > 5]       df.filter(df["col"] > 5)
  df.loc[]                df.filter()
  df.iloc[]               Pas d'équivalent             Pas d'index dans Spark
  ─────────────────────────────────────────────────────────────────────────


  TRANSFORMER LES DONNÉES
  ─────────────────────────────────────────────────────────────────────────
  Pandas                          PySpark                    Note
  ─────────────────────────────────────────────────────────────────────────
  df.groupby("col")               df.groupBy("col")          B majuscule
  df.sort_values("col")           df.orderBy("col")
  df["col"].apply(func)           df.withColumn("col", func)
  df["nouvelle_col"] = valeur     df.withColumn("nouvelle_col", valeur)
  ─────────────────────────────────────────────────────────────────────────


  FONCTIONS MATHÉMATIQUES — REMPLACENT NUMPY
  ───────────────────────────────────────────
      from pyspark.sql.functions import sqrt, log, exp, abs, mean, sum, min, max, count

      df.select(mean("col")).show()
      df.select(sqrt("col")).show()
      df.withColumn("log_col", log(df["col"])).show()


  VISUALISATION — RÈGLE UNIVERSELLE
  ───────────────────────────────────
  Spark ne sait pas dessiner. Toujours réduire avec Spark d'abord,
  puis passer à Pandas uniquement pour l'affichage.

      import matplotlib.pyplot as plt

      # BONNE PRATIQUE — Spark réduit, Pandas affiche
      df.select("ma_colonne") \
        .dropna() \
        .sample(fraction=0.05) \
        .toPandas()["ma_colonne"].hist()
      plt.show()

      # MAUVAISE PRATIQUE — ramène tout en mémoire → risque de crash
      df.toPandas().hist()

  toPandas() transfère les données du cluster vers la RAM locale.
  A n'utiliser qu'après avoir réduit le volume avec Spark.


  MACHINE LEARNING — REMPLACE SCIKIT-LEARN
  ──────────────────────────────────────────────────────────────────────────
  Scikit-learn                              PySpark MLlib
  ──────────────────────────────────────────────────────────────────────────
  from sklearn.linear_model import          from pyspark.ml.regression import
    LinearRegression                          LinearRegression

  from sklearn.ensemble import              from pyspark.ml.classification import
    RandomForestClassifier                    RandomForestClassifier

  from sklearn.preprocessing import        from pyspark.ml.feature import
    StandardScaler                            StandardScaler

  train_test_split(df, 0.8)                 df.randomSplit([0.8, 0.2])
  model.fit(X, y)                           model.fit(df)
  model.predict(X)                          model.transform(df)
  ──────────────────────────────────────────────────────────────────────────


  SQL — AUCUN CHANGEMENT DE SYNTAXE
  ────────────────────────────────────
      # Étape 1 — enregistrer le dataframe comme table (une seule fois)
      df.createOrReplaceTempView("ma_table")

      # Étape 2 — SQL classique
      spark.sql("SELECT * FROM ma_table LIMIT 10").show()
      spark.sql("SELECT pays, AVG(ventes) FROM ma_table GROUP BY pays").show()
      spark.sql("""
          SELECT col1, col2
          FROM ma_table
          WHERE annee = 2024
          ORDER BY col1 DESC
      """).show()

  La table est temporaire — elle existe uniquement pendant la session Jupyter.
  Si le kernel est redémarré, rappeler createOrReplaceTempView().
  Les données sur HDFS ne sont jamais affectées.


------------------------------------------------------------------------------
  PYSPARK.PANDAS — MIGRER SES NOTEBOOKS EXISTANTS
------------------------------------------------------------------------------

      # AVANT — Pandas classique
      import pandas as pd
      df = pd.read_csv("fichier.csv")

      # APRÈS — une seule ligne change, tout le reste est identique
      import pyspark.pandas as ps
      df = ps.read_csv("hdfs://namenode:9000/data/fichier.csv")

      df.head()                  identique
      df.describe()              identique
      df.groupby("col").mean()   identique
      df.dropna()                identique

  Limitation : l'ordre des lignes n'est pas garanti. Spark distribue les
  données sur plusieurs machines. Utiliser df.sort_values("col") pour
  forcer un ordre précis.

  Différence avec toPandas() :

      toPandas()         quitte Spark, entre dans Pandas
                         données rapatriées dans la RAM locale
                         Spark ne calcule plus rien

      pyspark.pandas     reste dans Spark du début à la fin
                         données restent distribuées sur le cluster
                         Spark fait tous les calculs


------------------------------------------------------------------------------
  CE QUE SPARK FAIT ET NE FAIT PAS
------------------------------------------------------------------------------

      Spark fait                      Spark ne fait pas
      ──────────────────────────────  ─────────────────────────────
      Nettoyer les données            Dessiner des graphiques
      Organiser et trier              Calculs NumPy matriciels
      Filtrer                         Deep learning natif
      Agréger et grouper              (TensorFlow, PyTorch)
      Joindre des tables
      Entraîner des modèles ML
      Requêtes SQL distribuées


------------------------------------------------------------------------------
  QUAND UTILISER QUOI
------------------------------------------------------------------------------

      Taille des données    Outil recommandé
      ─────────────────────────────────────────────────────────────
      < 1 Go            →   Pandas classique
                            plus simple, plus rapide

      1 Go à 5 Go       →   pyspark.pandas
                            syntaxe Pandas, puissance Spark

      > 5 Go            →   PySpark natif
                            contrôle total, meilleures performances
      ─────────────────────────────────────────────────────────────


------------------------------------------------------------------------------
  RÉFÉRENCE DES URLS ET PORTS
------------------------------------------------------------------------------

      URL                       Service                  Mot de passe
      ──────────────────────────────────────────────────────────────
      http://localhost:9870     Interface web HDFS        —
      http://localhost:8080     Interface web Spark       —
      http://localhost:8081     Spark Worker 1            —
      http://localhost:8082     Spark Worker 2            —
      http://localhost:8083     Spark Worker 3            —
      http://localhost:8888     Jupyter Notebook          bonjour
      ──────────────────────────────────────────────────────────────

==============================================================================