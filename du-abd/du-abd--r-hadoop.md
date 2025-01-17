# R et Hadoop

Ce document est une courte présentation de l'utilisation de R dans l'environnement Hadoop, basé sur une machine virtuelle proposée par [Hortonworks](https://fr.hortonworks.com/).

## Informations

Pour accéder à votre machine virtuelle, vous devez prendre l'IP entre `10.200.0.20` et `10.200.0.50`. Celles-ci ne sont accessibles que depuis l'IUT.

Il est possible d'accéder aux différentes applications en modifiant le port d'accès (sous la forme `http://ip:port`). Voici les différentes applications et les ports correspondant :

| Application             | Port |
|-------------------------|-----:|
| Accueil                 | 8888 |
| Ambari                  | 8080 |
| shell-in-a-box          | 4200 |
| Zeppelin                | 9995 |
| Resource manager Hadoop | 8088 |
| RStudio                 | 8090 |

Nous allons nous baser sur un tutoriel proposé par [Hortonworks](https://fr.hortonworks.com/tutorial/), permettant d'appréhender l'éco-système d'[Hadoop](http://hadoop.apache.org/) via leur machine virtuelle téléchargeable gratuitement ([ici](https://fr.hortonworks.com/products/sandbox/)).

## Retards aériens avec SparkR

Le document original est disponible à cette page : [Retards aériens avec SparkR](https://fr.hortonworks.com/tutorial/predicting-airline-delays-using-sparkr/)

### Installation de RStudio Server

Pour pouvoir suivre celui-ci, il faut d'abord installer RStudio sur la machine, vous devez procéder aux étapes suivantes

1. Aller sur le **shell-in-a-box** (port 4200) 
1. Se connecter avec le compte : `raj_ops/raj_ops`
1. Installer le lgogiciel R
```
sudo yum install R
```
1. Télécharger la version de **RStudio Server** :
```
wget https://download2.rstudio.org/rstudio-server-rhel-1.1.463-x86_64.rpm
```
1. Puis l'installer :
```
sudo yum install rstudio-server-rhel-1.1.463-x86_64.rpm
```
1. Puisque la machine virtuelle ne laisse pas accessible certains ports, il faut reconfigure le serveur. Pour cela, éditer le fichier de configuration 
```
sudo vi /etc/rstudio/rserver.conf
```
1. Appuyer sur la touche **`i`** (pour insérer du texte), puis sur la dernière ligne, écrire :
```
www-port = 8090
```
1. Faire `Esc`, puis taper `:wq` puis `Entrée`
1. Pour vérifier, taper la commande suivante (première ligne après le `$`). Le résultat devrait être le même que ci-dessous (2ème et 3ème lignes)
```
[raj_ops@sandbox-hdp ~]$cat /etc/rstudio/rserver.conf
# Server Configuration File
www-port = 8090
```
1. Vérifier l'installation :
```
sudo rstudio-server verify-installation
```
1. Arrêter puir démarrer le serveur :
```
sudo rstudio-server stop
sudo rstudio-server start
```

### Récupérer les données

Chaque année, environ 20% des vols sont soit retardés, soit annulés, avec comme impact un coût significatif à la fois pour les voyageurs et pour les compagnies. Comme exemple, nous allons construire un modèle supervisé qui prédit les les retards à partir de données précédentes. [Télécharger les données](https://raw.githubusercontent.com/hortonworks/data-tutorials/master/tutorials/hdp/advanced-analytics-with-sparkr-in-rstudio/assets/airline_data.zip) à partir di lien. Celles-ci incluent les informations des vols aux Etats-Unis duant l'année 2015. Chaque ligne contient 16 attributs :

- Year
- Month
- Day of Month
- Day of Week
- Flight Number
- Origin
- Destination
- Departure Time
- Departure Delay
- Arrival Time
- Arrival Delay
- Cancelled
- Cancellation Code
- Air Time
- Distance

Après avoir récupérer les données, décompresser les et charger les fichiers `train_df.csv` et `test_df.csv` dans le répertoire `/tmp` de HDFS en utilisant la vue Fichier (`Files View`). Utiliser **Ambari** (port 8080) pour cela, avec le compte  `amy_ds`/`amy_ds`. 

### Préparer SparkR dans RStudio

Maintenant, nous pouvons nous connecter à **RStudio** (port 8090) toujours en utilisant le compte `amy_ds`. Nous devons créer un objet, nommé ici `SparkContext`, qui va connecter R au cluster. Pour cela, nous utilisons la fonction `sparkR.init()`. De la même manière, nous devons créer un objet, nommé `SqlContext`, pour travailler sur les data frames créés à partir de `SparkContext`.

Tout d'abord, nous créons une variable d'environnement `SPARK_HOME` qui contient la localisation des librairies Spark. Nous chargerons le package SparkR et ferons appel à la fonction d'initialisation pour créer le contexte. Nous ajoutons aussi des informations sur les drivers Spark et le package csv, pour que SparkR puisse lire les fichiers csv.

Ecrire et exécuter ces lignes de commandes dans RStudio :

```
Sys.setenv(SPARK_HOME = "/usr/hdp/current/spark2-client")
library(SparkR, lib.loc = c(file.path(Sys.getenv("SPARK_HOME"), "R", "lib")))
sc <- sparkR.session(
  master = "local[*]", 
  sparkEnvir = list(spark.driver.memory = "2g"),
  sparkPackages = "com.databricks:spark-csv_2.10:1.4.0")
sqlContext <- sparkR.session()
```

### Préparation des données d'apprentissage

Pour plus d'informations sur l'API SparkR, vous pouvez consulter [cette page](https://spark.apache.org/docs/latest/api/R/index.html). Il est possible de créer un data frame SparkR à partir d'un data frame R local, de sources de type csv ou autres, ou de tables **Hive**.

Nous allons utiliser la fonction `read.df()` pour lire un fichier à partir d'une source (**HDFS** ici) et créer une data frame SparkR. 

Exécuter cette ligne pour créer le jeu de données à partir du fichier `/tmp/train_df.csv`, les en-têtes étant incluses :

```
train_df <- read.df("/tmp/train_df.csv", "csv", header = "true", inferSchema = "true")
```

Vous devriez pour voir les premières lignes du jeu de données avec la commande `head(train_df)`. On peut aussi examiner le jeu ainsi créé dans la fenètre d'environnement.

Maintenant, nous allons créer des nouvelles colonnes pour notre future analyse.

On commence par indiquer si le vol est le week-end ou non avec une indicatrice binaire (1 si c'est un vendredi, samedi ou dimanche, et 0 sinon).

```
train_df$WEEKEND <- ifelse(train_df$DAY_OF_WEEK == 5 | train_df$DAY_OF_WEEK == 6 | train_df$DAY_OF_WEEK == 7, 1, 0)
```

Ensuite, nous créons une nouvelle colonne nommée `DEP_HOUR` qui récupère l'heure de départ de la colonne `DEP_TIME`.

```
train_df$DEP_HOUR <- floor(train_df$DEP_TIME / 100)
```

Nous créons maintenant une autre indicatrice, nommée `DELAY_LABELED`, qui va prendre la valeur 1 si le délai est supérieur à 15 minutes et 0 sinon.

```
train_df$DELAY_LABELED <- ifelse(train_df$ARR_DELAY > 15, 1, 0)
train_df$DELAY_LABELED <- cast(train_df$DELAY_LABELED,"integer")
```

Nous allons garder les vols qui non pas été annulés, en filtrant sur la valeur de la variable `CANCELLED` (qui doit être égale à 0 donc).

```
train_df <- train_df[train_df$CANCELLED == 0,]
```

La prochaine étape de nettoyage concerne les données manquantes. En regardant les données, on peut voir qu'il y a beaucoup de `NA` dans la colonne `ARR_DELAY`. Nous gardons que les lignes pour lesquelles nous avons l'information.

```
train_df <- dropna(train_df,cols = 'ARR_DELAY')
```

Maintenant, nous pouvons voir le schéma des données du data frame SparkR, avec la commande suivante :

```
printSchema(train_df)
```

Pour pouvoir faire le travail par la suite, il nous est nécessaire de transformer les variables `ARR_DELAY` and `DEP_DELAY` en entier.

```
train_df$ARR_DELAY <- cast(train_df$ARR_DELAY,"integer")
train_df$DEP_DELAY <- cast(train_df$DEP_DELAY,"integer")
```

On peut regarder de nouveau le jeu de données (les premières lignes) poru vérifier que tout et ok.

### Analyse exploratoire

<!--
At the end of this tutorial, we will be able to predict which flight is likely to be delayed. We can classify our dataset into two values- 0 or 1 (0 for flights on time and 1 for flights delayed). But before creating a model, let us visualize the data what we have right now.
-->

Nous allons explorer visuellement ces données pour avoir un apercu des causes des retards.


#### Retards

Nous créons un agrégat sur la base de la colonne `DELAY_LABELED`, pour compter le nombre de vols retardés et de vols à l'heure. Nous utilisons la fonction d'aggrégation de SparkR pour cela.

```
delay <- agg(group_by(train_df, train_df$DELAY_LABELED), count = n(train_df$DELAY_LABELED))
```

On introduit une nouvelle colonne nous permettant de définir un label pour chaque situation.

```
delay$STATUS <- ifelse(delay$DELAY_LABELED == 0, "ontime", "delayed")
```

Puis, nous allons convertir notre data frame SparkR en data frame R classique, pour visualiser les données avec `ggplot2` (tout en supprimant la première colonne qui ne nous est plus utile ici).

```
delay_r <- as.data.frame(delay[,-1])
```

Nous allons ajouter le pourcentage pour pouvoir l'afficher dans le graphique.

```
delay_r$Percentage <- (delay_r$count / sum(delay_r$count)) * 100
delay_r$Percentage <- round(delay_r$Percentage,2)
```

Maintenant, nous allons installer et charger le package `ggplot2` (très bonne librairie de visualisation de données, basée sur la grammaire des graphiques).

```
install.packages("ggplot2")
library(ggplot2)
```

Nous pouvons donc afficher maintenant le diagramme circulaire, nous montrant le pourcentage de vols retardés et à l'heure.

```
ggplot(delay_r, aes(x = "", y = Percentage, fill = STATUS)) +
  geom_bar(stat = "identity", width = 1, colour = "black") +
  coord_polar(theta = "y", start = 0) + 
  ggtitle("Pie Chart for Flights") + 
  theme(axis.text.x = element_blank()) + 
  geom_text(aes(y = Percentage/2, label = paste0(Percentage,"%"), hjust=2))
```

Ce graphique indique que 18.17% des vols sont retardés.

#### Inflluence du jour de la semaine

Nous voulons maintenant regarder l'impact du jour de la semaine sur les retards. Pour cela, nous allons créer 2 jeux de données (`delay_flights` et `non_delay_flights`) contenant les détails pour chaque situation.

```
delay_flights <- filter(train_df,train_df$DELAY_LABELED == 1)
non_delay_flights <- filter(train_df,train_df$DELAY_LABELED == 0)
```

Puis, nous réalisons l'aggrégat selon le jour de la semaine pour chaque data frame.

```
delay_flights_count <- agg(
  group_by(delay_flights,delay_flights$DAY_OF_WEEK), 
  DELAY_COUNT = n(delay_flights$DELAY_LABELED)
)
non_delay_flights_count <- agg(
  group_by(non_delay_flights,non_delay_flights$DAY_OF_WEEK), 
  NON_DELAY_COUNT = n(non_delay_flights$DELAY_LABELED)
)
```

Nous allons maintenant joindre ces deux data frames.

```
dayofweek_count <- merge(
  delay_flights_count, 
  non_delay_flights_count, 
  by.delay_flights_count = DAY_OF_WEEK, 
  by.non_delay_flights_count = DAY_OF_WEEK)
```

Lors de la jointure, la colonne ayant servie à la faire est duppliquée. Nous la supprimons donc une des deux. Puis nous renommons la colonne restante.

```
dayofweek_count$DAY_OF_WEEK_y <- NULL
dayofweek_count <- withColumnRenamed(dayofweek_count, "DAY_OF_WEEK_x", "DAY_OF_WEEK")
```

Nous convertissons ce data frame SparkR en data frame R.

```
dayofweek_count_r <- as.data.frame(dayofweek_count)
```

Nous introduisons 2 nouvelles colonnes (`Delayed` et `Ontime`) qui contiennent les pourcentages respectifs pour les retards et les vols à l'heure.

```
dayofweek_count_r$Delayed <- (dayofweek_count_r$DELAY_COUNT/(dayofweek_count_r$DELAY_COUNT + dayofweek_count_r$NON_DELAY_COUNT)) * 100
dayofweek_count_r$Ontime <- (dayofweek_count_r$NON_DELAY_COUNT/(dayofweek_count_r$DELAY_COUNT + dayofweek_count_r$NON_DELAY_COUNT)) * 100
dayofweek_count_r <- dayofweek_count_r[,-2:-3]
```

Puis nous ajoutons la colonne qui va représenter le ratio de vols retardés par rapport aux vols à l'heure.

```
dayofweek_count_r$Ratio <- dayofweek_count_r$Delayed/dayofweek_count_r$Ontime * 100
dayofweek_count_r$Ratio <- round(dayofweek_count_r$Ratio,2)
```

En regardant le jeu de données, nous voyons qu'il est dans un format dit large (`wide` : une ligne par observation, chaque mesure dans une colonne). Pour le représenter avec `ggplot2`, nous devons le tranformer au format long (une ligne par observation et par mesure). Nous pouvons utiliser la fonction `melt()` du package `reshape2` pour cela.

```
library(reshape2)
DF1 <- melt(dayofweek_count_r, id.var="DAY_OF_WEEK")
DF1$Ratio <- DF1[15:21,3]
```

Regarder maintenant le jeu de données ainsi créé.

```
DF1
```

Nous faisons quelques modifications de ce jeu pour pouvoir réaliser un graphique propre.

```
DF1 <- DF1[-15:-21,]
DF1[8:14,4] <- NA
```

Pour réaliser un diagrammeen en barres,avec l'information sur le ratio, exécuter les commandes suivantes :

```
install.packages("ggrepel")
library(ggrepel)
ggplot(DF1, aes(x = DAY_OF_WEEK, y = value, fill = variable)) + 
  geom_bar(stat = "identity") + 
  geom_path(aes(y = Ratio, color = "Ratio of Delayed against Non Delayed")) + 
  geom_text_repel(aes(label = Ratio), size = 3) + 
  ggtitle("Percentage of Flights Delayed") + 
  labs(x = "Day of Week", y = "Percentage")
```

Comme vous pourvez le voir, les retards sont plus fréquents le lundi et le jeudi, avec une baisse le week-end (mais une reprise le dimanche).

#### Influence de la destination

Nous allons reprendre le même schéma pour regarder l'influence de la destination sur les retards.

Créons deux aggrégats pour les retardés et les vols à l'heure, en fonction des destinations, en se concentrant sur certains aéroports (LAX, SFO, HNL, PDX).

```
destination_delay_count <- agg(
  group_by(delay_flights,delay_flights$DEST), 
  DELAY_COUNT = n(delay_flights$DELAY_LABELED))
destination_delay_count <- destination_delay_count[(destination_delay_count$DEST == "LAX" | destination_delay_count$DEST == "SFO" | destination_delay_count$DEST == "HNL" | destination_delay_count$DEST == "PDX") ,]

destination_non_delay_count <- agg(
  group_by(non_delay_flights,non_delay_flights$DEST), 
  NON_DELAY_COUNT = n(non_delay_flights$DELAY_LABELED))
destination_non_delay_count <- destination_non_delay_count[(destination_non_delay_count$DEST == "LAX" | destination_non_delay_count$DEST == "SFO") | destination_delay_count$DEST == "HNL" | destination_delay_count$DEST == "PDX" ,]
```

Puis regroupons ces deux data frames en 1 seul.

```
destination_count <- merge(
  destination_delay_count, 
  destination_non_delay_count, 
  by.destination_delay_count = DEST, 
  by.destination_non_delay_count = DEST)
destination_count$DEST_y <- NULL
destination_count <- withColumnRenamed(destination_count,"DEST_x","DEST")
```

Puis nous le convertissons en data frame R.

```
destination_count_r <- as.data.frame(destination_count)
```

Créons nos deux colonnes (`Delayed` et `Ontime`) qui vont contenir les valeurs des pourcentages.

```
destination_count_r$Delayed <- (destination_count_r$DELAY_COUNT/(destination_count_r$DELAY_COUNT+destination_count_r$NON_DELAY_COUNT)) * 100
destination_count_r$Ontime <- (destination_count_r$NON_DELAY_COUNT/(destination_count_r$DELAY_COUNT+destination_count_r$NON_DELAY_COUNT)) * 100
destination_count_r <- destination_count_r[,-2:-3]
```

Plus la colonne nommé `Ratio` qui va contenir la proportion de vols retardés.

```
destination_count_r$Ratio <- destination_count_r$Delayed/destination_count_r$Ontime * 100
destination_count_r$Ratio <- round(destination_count_r$Ratio,2)
```

Et comme précédemment, nous allons transformer notre data frame large en data frame long, pour pouvoir créer un diagramme en barres empilées.

```
DF2 <- melt(destination_count_r, id.var="DEST")
DF2$Ratio <- DF2[9:12,3]
DF2 <- DF2[-9:-12,]
DF2[5:8,4] <- NA
```

Puis créons notre diagramme.

```
ggplot(DF2, aes(x = DEST, y = value, fill = variable)) + 
  geom_bar(stat = "identity") + 
  geom_path(aes(y = Ratio, color = "Ratio of Delayed against Non Delayed"),group = 1) + 
  geom_text_repel(aes(label = Ratio), size = 3) + 
  ggtitle("Percentage of Flights Delayed by Destination") +
  labs(x = "Destinations", y = "Percentage")
```

Les petites villes de destination ont une ratio plus élevés de retards.

#### Influence de la ville d'origine

Réalisons la même petite étude pour les villes d'origine des vols. Nous créons nos deuxdata frames en se concentrant sur les aéroports SNA, ORD, JFK and IAH.

```
origin_delay_count <- agg(
  group_by(delay_flights,delay_flights$ORIGIN), 
  DELAY_COUNT = n(delay_flights$DELAY_LABELED))
origin_delay_count <- origin_delay_count[(origin_delay_count$ORIGIN == "SNA" | origin_delay_count$ORIGIN == "ORD" | origin_delay_count$ORIGIN == "JFK" | origin_delay_count$ORIGIN == "IAH") ,]
origin_non_delay_count <- agg(
  group_by(non_delay_flights,non_delay_flights$ORIGIN), 
  NON_DELAY_COUNT = n(non_delay_flights$DELAY_LABELED))
origin_non_delay_count <- origin_non_delay_count[(origin_non_delay_count$ORIGIN == "SNA" | origin_non_delay_count$ORIGIN == "ORD" | origin_delay_count$ORIGIN == "JFK" | origin_delay_count$ORIGIN == "IAH") ,]
```

Fusionnons les data frames et convertissons le résultat en un data frame R.

```
origin_count <- merge(origin_delay_count, origin_non_delay_count, by.origin_delay_count = ORIGIN, by.origin_non_delay_count = ORIGIN)
origin_count$ORIGIN_y <- NULL
origin_count <- withColumnRenamed(origin_count, "ORIGIN_x", "ORIGIN")
origin_count_r <- as.data.frame(origin_count)
```

Puis nous allons ajouter nos trois colonnes (`Delayed`, `Ontime` et `Ratio`).

```
origin_count_r$Delayed <- (origin_count_r$DELAY_COUNT/(origin_count_r$DELAY_COUNT+origin_count_r$NON_DELAY_COUNT)) * 100
origin_count_r$Ontime <- (origin_count_r$NON_DELAY_COUNT/(origin_count_r$DELAY_COUNT+origin_count_r$NON_DELAY_COUNT)) * 100
origin_count_r <- origin_count_r[,-2:-3]
origin_count_r$Ratio <- origin_count_r$Delayed/origin_count_r$Ontime * 100
origin_count_r$Ratio <- round(origin_count_r$Ratio,2)
```

Et comme précédemment encore, nous allons modifier le format du data frame pour réaliser notre diagramme en barres empilées.

```
DF3 <- melt(origin_count_r, id.var="ORIGIN")
DF3$Ratio <- DF3[9:12,3]
DF3 <- DF3[-9:-12,]
DF3[5:8,4] <- NA

ggplot(DF3, aes(x=ORIGIN,y=value,fill=variable)) + 
  geom_bar(stat = "identity") + 
  geom_path(aes(y = Ratio, color = "Ratio of Delayed against Non Delayed"), group = 1) + 
  geom_text_repel(aes(label = Ratio), size = 3) + 
  ggtitle("Percentage of Flights Delayed by Origin") +
  labs(x = "Origins", y = "Percentage")
```

A l'inverse, les petites villes (SNA ici) ont un plus petit ratio de retards.

## Pour aller plus loin

Il serait intéressant ici de faire un modèle (au choix entre régression logistique, arbre de décision, réseaux de neurones, ...) permettant de prédire le retard d'un vol en fonction des informations présentes.

Voici un autre tutoriel très intéressant pour vou :

[Introduction au Machine Learning avec Spark et Zeppelin](https://fr.hortonworks.com/tutorial/intro-to-machine-learning-with-apache-spark-and-apache-zeppelin/)


