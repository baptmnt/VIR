# TD8 - Intégration continue, livraison continue et déploiement continu

Objectif du TD :
- :dart: Avoir une vision de processus de déploiement : comment passe-t-on du code source à une application déployée ?
- :dart: Savoir mettre en place un pipeline gitlab pour automatiser ce processus

Ce TD assume que vous avez les prérequis suivants : 
- Savoir créer et interagir avec un dépôt git
- ???

Dans les TDs précédents, nous avons vu comment passer du code source d'une application au déploiement de cette application : 
- Build
- Test
- Delivery
- Deploy

Dans ce TD, nous allons voir comment automatiser ces étapes.

Bien qu'il soit possible d'automatiser ces tâches en local, il est commun d'effectuer ses opérations dans un gestionnaire de version distant comme Github ou Gitlab. En effet, pour valider des modifications de code, il est plus simple de systématiquement vérifier que tout fonctionne + 

Rappel rapide : Git vs Gitlab

Dans notre cas, nous utiliserons Gitlab pour mettre en place cette automatisation. L'objectif est le suivant: chaque nouveau commit poussé vers le dépôt gitlab entraine l'exécution d'une *pipeline*. Une pipeline est une ensemble de tâches - *jobs* en anglais - qui vont entrainer la compilation, le test, voir le déploimenent du code modifié. 

Ces tâches sont regroupées en étapes - *stages* en anglais -. Les étapes par défaut dans Gitlab sont `build`, `test` et `deploy`, mais il est possible de définir des étapes personnalisées. Les tâches au sein d'une même étape s'exécutent en parallèle. Les étapes s'exécutent les unes après les autres : les tâches de `test` ne s'exécuteront qu'une fois les tâches de `build` terminées. Si notre programme C ne compile pas (étape `build`),  alors on ne va pas le tester (étape `test`).

La figure suivante présente la visualisation d'une *pipeline* gitlab avec deux *stages* : `build` et `test`. Le stage `build` contient un seul *job*, tandis que le stage `test` en contient deux. 

![Pipeline Menu](/figures/ci-status.png)

On retrouve ici deux étapes ou *stages* - `build` et `test` -, et leur tâches ou *jobs*. Dans notre cas, les tâches sont dans trois status différents :
- JobA a été complété (status `success`)
- JobB est en cours d'exécution (status `running`)
- JobC est en attente d'exécution (status `pending`)

Dans la suite du TD, nous détaillons comment mettre en place un pipeline avec Gitlab CI/CD. Les concepts évoqués seront transposables à d'autres outils de CI/CD (Github Actions, Jenkins, CircleCI), même si leur implémentation varie.

## Pipelines Gitlab

La mise en place d'un pipeline Gitlab ce fait via un fichier déposé à la racine du dépôt : `.gitlab-ci.yml`. Ce fichier décrit les étapes et les tâches à exécuter au format `yaml`. Le `.gitlab-ci.yml` correspondant à la pipeline visible au dessus est le suivant :

```yaml
# Fichier .gitlab-ci.yml
jobA:
  stage: build
  image: python:3.14
  script:
    - echo "Hi $GITLAB_USER_LOGIN!, running JobA"
  
jobB:
  stage: test
  script:
    - echo "Testing something..."
    - ping -c 2 8.8.8.8

jobC:
  stage: test
  script:
    - echo "Testing nothing"
    - cat README.md
```

Chaque job exécute une suite de commandes.

- Parler des runners, que les jobs s'exécutent dans docker
- Ici exemple minimal, mais plein d'autres paramètres. Variables $GITLAB_USER_LOGIN

## Exécution d'un pipeline

On va prendre l'exemple de ce dépôt : 
- https://gitlab.insa-lyon.fr/hreymond/cicd_example

Sur la page gitlab de votre dépôt, vous devriez observer votre commit, avec un petit badge bleu, ou vert :

![Commit avec badge CI/CD](/figures/ci-cd_badge.png)

Ce badge indique la réussite de la pipeline associée au commit. 
En cliquant sur ce badge, vous retrouvez le détail du pipeline et l'état des tâches.

Cliquer sur le JobA pour voir ces logs. Chaque tâche exécutée suit le même processus :
- Un conteneur docker est créé à partir d'une image par défaut ou spécifiée avec le paramètre `image:`
- Le contenu du dépôt gitlab correspondant au commit qui a déclenché la pipeline est copié dans le conteneur.
- L'ensemble des commandes spécifiées dans le Job sont exécutées.

:question: Saurez-vous retrouver dans les logs :
- Le nom de l'image docker utilisée pour exécuter le job ?
- Le hash du commit qui est utilisé par le build ?
- La commande exécutée ?

On va partir de cet exemple pour mettre en place notre CI.
Fork -> Présenter étapes pour fork + principe d'une fork

# Partie 1 - Intégration continue (CI)

Mettre en place un CI pour un truc simple : l'app SuperDB

L'intégration continue vise à ce que les modification du code source d'un logiciel soient vérifiées.

Pour tester un logiciel, il est nécessaire d'avoir une première étape, dite de "build" : compilation des dépendances

## Build 

Supprimer les Jobs existants dans `.gitlab-ci.yml`, et créez un nouveau job nommé `build`. Ce job doit compiler `superDB.c` pour créer `superDBEXE`.

On utilisera directement gcc : `gcc -o superDBEXE superDB.c`
Pour cela, deux approches :
- Prendre une image générique : `debian`, `alpine` et installer les dépendances `gcc libc6-dev`
- Prendre une image spécialisée : `gcc`



Compléter le Job `build` dont le template est fourni ci-dessous. Ce Job doit concevoir l'image docker à partir du `Dockerfile` présent dans le dossier `website`. Tagger cette image `website:v3` (cf [TD2](/TD2-docker/)). 

```yaml
build:
  # L'image choisie pour exécuter les commandes de notre Job
  image: docker:24.0.5-cli 
  # L'ensemble des instructions à exécuter avant les commandes 
  # de notre Job. Ici, on se déplace dans le dossier "website"
  before_script : 
  - cd website
  # Cette ligne permet d'avoir accès au moteur de conteneur docker
  # On reviendra sur cette partie plus tard dans le TD
  services:
  - docker:24.0.5-dind
  # Les commandes à éxécuter pour concevoir notre image
  steps:
  - echo "Building website image !"
  - ...
```

Cette étape nous permet de vérifier que notre image docker se construit bien.

git add, commit, push : verifiez que votre Job est bien exécuté, et que l'image se construit sans problème.

## Test

Maintenant que notre image est bien construite, on veut tester les fonctionnalités de notre site.

Copier la définition du Job `build` pour créer le job `test`. 
Étendre ce job pour :
- lancer un conteneur à partir de l'image `website:v3`, en mode démon (`-d`) que vous appelerez `website`
- attendre 5 seconde que le serveur se lance (commande `sleep`)
- exécuter la commande `python test_website.py` à l'intérieur du conteneur `website`, avec `docker exec`. Ce petit script python va vérifier que le serveur réponde au requêtes HTTP.

git add, commit, push : verifiez que votre Job est bien exécuté, et que l'image est testée sans problème.

Changer le Dockerfile du site web pour que la commande lancée au démarrage ne soit plus `flask run --host=0.0.0.0` mais `echo coucou`. 

git add, commit, push. 

:question: Votre pipeline de test permet-elle de repérer la perte de fonctionnement de votre conteneur ?

## Cache

Dans nos deux Job : `build` et `test`, nous construisons l'image `website`, pas très malin. Comment faire en sorte que `build` produise une image qui sera ensuite utilisée par `test` ? À l'aide de *caches*



## Cache vs Artifact

## Artifact 

## Services



# CD : Livraison continue



# CD (encore ?) : Déploiement continu


# Liens utiles

- [Formation CI/CD](https://blog.stephane-robert.info/docs/pipeline-cicd/gitlab/)

Pour référence, une liste de tout les états possibles est disponible [ici](https://docs.gitlab.com/ci/jobs/#available-job-statuses)