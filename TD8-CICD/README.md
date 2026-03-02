# VIR_CICD

CI/CD : Quezaquo ? 

## Pipelines Gitlab

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

Trois Jobs, deux stages. 
Stage, Spécificité de gitlab, étape
 - Tout les jobs d'une étape doivent être finis avant de pouvoir entamer la suivante

Chaque job exécute une suite de commandes.

Git add, commit, push

Sur la page gitlab de votre dépôt, vous devriez observer votre commit, avec un petit badge bleu, ou vert :
![Commit avec badge CI/CD](/figures/ci-cd_badge.png)

En cliquant sur ce badge, vous retrouvez le détail de votre pipeline

![Pipeline Menu](/figures/ci-status.png)

On retrouve ici les deux étapes ou *stages* définis précédemment, `build` et `test`, et leur tâches ou *jobs*. Dans notre cas, les tâches sont dans trois status différents :
- JobA a été complété (status `success`)
- JobB est en cours d'exécution (status `running`)
- JobC est en attente d'un runner (status `pending`)

Pour référence, une liste de tout les états possibles est disponible [ici](https://docs.gitlab.com/ci/jobs/#available-job-statuses)

Cliquer sur une des tâches complétées pour voir les logs. Chaque tâche exécutée suit le même processus :
- Un conteneur docker est créé à partir d'une image par défaut ou spécifiée
- Le contenu du dépôt gitlab correspondant au commit en question est copié dans le conteneur.
- L'ensemble des commandes spécifiées dans le Job sont exécutées.

:question: Saurez-vous retrouver dans les logs :
- Le nom de l'image utilisée pour exécuter le job ?
- Le hash du commit qui est utilisé par le build ?
- La commande exécutée ?

# CI : Intégration continue

Retour du site du TD3 dans `website`

L'intégration continue vise à ce que les modification du code source d'un logiciel soient vérifiées.

Pour tester un logiciel, il est nécessaire d'avoir une première étape, dite de "build" : compilation des dépendances

## Build 

Créer un nouveau Job, appelé `build`, qui conçoit l'image docker à partir du `Dockerfile` dans `website`. Tagger cette image `website:v3`

```yaml
build:
  image: docker:24.0.5-cli
  before_script : 
  - cd website
  services:
  - docker:24.0.5-dind
```

Cette étape nous permet de vérifier que notre image docker se construit bien.

git add, commit, push : verifiez que votre Job est bien exécuté, et que l'image se construit sans problème.

## Test

Maintenant que notre image est bien construite, on veut tester les fonctionnalités de notre site.

Copier la définition du Job `build` pour créer le job `test`. 
Étendre ce job pour :
- lancer un conteneur à partir de l'image `website:v3`, en mode démon (`-d`)
- attendre 5 seconde que le serveur se lance (commande `sleep`)
- vérifier qu'il est possible de se connecter au serveur à l'aide de la commande `curl localhost:5000`

## Cache

## Cache vs Artifact

## Artifact 

# CD : Livraison continue


# CD (encore ?) : Déploiement continu