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
Il faut tout de même les dépendances (`gcc`, `libc6-dev`). 

### Sélectionner une image adaptée au job

Pour cela, deux méthodes :
- Prendre une image générique : `debian`, `alpine` et installer les dépendances `gcc libc6-dev`
- Prendre une image spécialisée : `gcc`

Git add , Git commit, Git push

## Test

Copier votre Job `build` pour créer le job `test`.
Étendre ce job pour appeler le script `testSuperDB.sh`. Ce script permet de tester le fonctionnement de la base de donnée SuperDB.

ga; gc; gp -> Est-ce que tout marche ?

Git add , Git commit, Git push

## Cache

Problème : on fait deux fois la compilation de SuperDB.

Mise en cache de superDBExe

```yaml
job:
  cache:
    - key :
      paths :
      - <Chemin à sauvegarder>
      - <Chemin à sauvegarder>
```

Ajouter un cache aux jobs `build-db` et `test-db`, pour le chemin `SuperDB/superDBExe`.


### Code coverage

Code coverage c'est quoi ? 

Test, et on vérifie que l'on a testé toutes les branches possibles de notre programme.

Nouveau Job : `coverage`. 

On compilera l'appli dans coverage car il faut build l'appli avec des paramètres particulier 
Modifier la compilation de super db et ajouter les arguments suivants `--coverage -g -O0`

De cette manière, la compilation de SuperDB va génerer un fichier `.gcno`. Quand on va exécuter nos tests de SuperDB, ça va générer un fichier `.gcda`

On va utiliser `covr`, un outil pour analyser ces traces pour langage C.
Modifier le job pour installer le paquet apt `covr` (attention à bien faire un `apt update` avant)

covr -> Normalement on a le pourcentage

Ga; Gc; Gp

vérifier qu'il est bien visible dans les logs
-> Cool, mais personne ne va aller le voir
->Fail si pas assez de coverage

Ajouter l'option `--fail-under-line=90` à la commande `gcovr`

Maintenant, chaque ajout de fonctionnalité devra venir avec un test pour que la couverture de code soit au dessus de 90%. 

Ga; Gc; Gp

On observe que le test fail car pas assez de coverage. Cependant, on ne peut pas voir quelles sont les lignes qui ne sont pas empruntées. Ajouter l'option `--html-details coverage.html` permet de générer un rapport au format html indiquant les lignes de code non couvertes par les tests.

Problème : comment récupérer ce rapport ? 

On voudrait un moyen d'accéder aux fichiers HTML et CSS générés.

## Artifact 

Introduction : qu'est ce qu'un artefact ? 

Comme pour le `cache`, on indique les chemins que l'on souhaite exporter pour chaque job :

```yaml
artifacts:
  paths:
    - "*.html"
    - "*.css"
```

ga; gc; gp 

![Menu des artefacts](/figures/artifacts.png)

Télécharger et ouvrez les artefacts.

# Partie 2 - Livraison continue (CD)

Registries Gitlab 

- Générique (fichiers en tout genre)
- Docker 
- Helm


Registry qui écoutent sur l'API. Identification via un token auto-généré par chaque Job

Exemple : publier un ficher dans le registre générique [Doc](https://docs.gitlab.com/user/packages/generic_packages/?tab=With+a+Bash+script#publish-a-single-file)

To publish a single file, use the following API endpoint:

`PUT /projects/<id>/packages/generic/<package_name>/<package_version>/<file_name>`

Replace the placeholders in the URL with your specific values:

- id: Your project ID or URL-encoded path
- package_name: Name of your package
- package_version: Version of your package
- file_name: Name of the file you’re uploading. See valid package filename format below.

Créer un job "delivery", stage "deploy", qui exécute la commande suivante :

```bash
curl -v -X PUT --header "JOB-TOKEN: ${CI_JOB_TOKEN}" --upload-file superDBExe https://gitlab.insa-lyon.fr/api/v4/projects/${CI_PROJECT_ID}/packages/generic/superDB/latest/superDBExe
```

:question: À quoi correspondent chacune des options de cette commande curl ?

Retrouver les paramètres dans l'url ?

- Choisir une image adaptée (cf [Sélectionner une image adaptée au job](#sélectionner-une-image-adaptée-au-job))
- Configurer le cache pour récupérer le binaire `superDBExe` 
- ajouter la commande

ga; gc; gp

-> Version harcodée, pas fou
-> Pas forcément besoin de stocker une version du logiciel par commit

### Version manuelle

ajouter `when: manual` au job pour qu'il faille le démarrer manuellement.

L'image de Rancher ne fonctionne pas, et la hardened image non plus

TODO:

- [ ] Regarder des uses-case de key pour le cache

# Partie 3 - Images docker

Cas de docker un peu particulier, car pour build une image docker, il nous faut un deamon docker, capable de traiter les commandes `docker build .`



## Build docker 

Notre binaire est utilisé uniquement par le job suivant, mais certaines productions de la CI/CD sont intéressantes

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