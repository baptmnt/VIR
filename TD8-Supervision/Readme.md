Nous allons installer une infrastructure de supervision minimale. Elle surveille l'application Web minecraft que vous avez précédemment développé.
Dans votre souvenir, elle passe par traefik pour l'accès externe et nous avons mis à disposition sur la clé le chart elm permettant de l'installer directement.

L'application Web s'accède selon deux routes : 
 / --> Qui demande la saisie d'un nom d'avatar  
 /display_skin?username=toto --> Qui requête une API externe, récupère l'avatar et met à jour un compteur dans un microservice de base de données. 

Nous testerons principalement la seconde route d'accès.   


L'infrastructure minimale standard de supervision repose sur les composants et l'architecture suivante.  

- **Traefik** est le gestionnaire d'accès de type API Gateway. C'est le composant que nous voulons surveiller. 
  - Il expose des métriques via la route localhost:9100/metrics
- Cette route est régulièrement interrogée par le gestionnaire de sonde **Prometheus**.
  - Il peut être accédé via une route localhost:9090
- Les valeur collectées et aggrégées par prometheus peuvent être visualisées graphiquement par **graphana** 
  - La visualisation peut être accédée par la route localhost:3000

metrics --> prometheus --> graphana, sont les éléments les plus traditionnels des infrastructures de supervision de système. Le nom des composants peuvent changer, mais les trois rôles clés : data -> aggregation -> visualisation existent toujours. 
Nous vous proposons la mise en place de cette infrastructure, puis la vérification via l'usage de l'application web minecraft.

**Prometheus** n'est pas lié à Kubernetes, c'est un outil de collecte et de stockage de metriques dans des bases de données temporelles. Sont role est d'absorber des grandes charges de données les indexer, les normaliser et permettre leur intérogations selon n'importe quel critère souhaité. Les outils comme elasticsearch, opentelemetry ou prometheus servent le même but : collecter et mettre à disposition des centre d'administration des sondes et des alarmes de surveillance des infrastructures.

Il existe un chart elm de prometheus chargé de la supervision d'infrastructure kube. Ce chart 'prometheus-community/kube-prometheus' est installé via le script `startPrometheus.sh'. La version que nous avons configuré n'install pas les sous-charts 'kubeStateMetrics' et nodeExporter' qui permettent de remonter toutes les metriques d'état du cluster et d'état des noeuds. Nous réduisons ces données afin que vous vous focalisiez sur les metriques de 'traefik'. 

Pour l'installation, nous vous suggérons de lancer une console `k9s` dans une fenêtre afin d'observer la progression de l'installation. 

`./startPrometheus.sh`

Vous pouvez vérifier que le service fonctionne en demandant les metriques actuellement scrappées. Si votre port-forward tourne. 

`curl http://localhost:9090/api/v1/label/__name__/values | jq`
Normalement vous ne voyez pas de métriques liées à Traefik.

:question: Pouvez-vous faire le même contrôle sans le port-forward activé ?   
:question: Pouvez-vous accéder à ce port-forward d'une autre machine de la salle ? Tester avec le navigateur de votre voisin. (ps : la réponse est oui)

Prometheus est bien installé. Il vient accompagné de graphana que nous lançons plus tard, et des 'servicesmonitor' pour découvrir les sondes/metriques accessibles.


# Installation de Traefik
Les metriques sont accessible sur les noeuds par la route `<ip:9100>/metrics` Traefik n'échappe pas à la règle. Il peut participer à l'emission de metriques pour prometheus à l'instalation. Nous avons une version modifiée du lancement de traefik sur la clé que vous pouvez lancer. `./startPrometheusTraefik.sh`. Cette version se charge de valider 3 choses :   
  --set metrics.prometheus.enabled=true --> Autorise la récolte des metrics par prometheus  
  --set metrics.prometheus.service.enabled=true --> Crée un service http annoncant la route /metrics
  --set metrics.prometheus.serviceMonitor.enabled=true --> Permet à prometheus de découvrir le moniteur de traefik.

Toutefois le moniteur de traefik n'est pas découvert, il faut une positionner un label spécifique dans la description. (Il s'agit certainement d'un bug d'install).
La ligne du script `kubectl label servicemonitor traefik release=prometheus` se charge de cela.


Vous pouvez maintenant tester votre installation  : 
- Mise à disposition de la route par traefik : `kubectl get services`, puis `curl <xxx.yyy>:9100/metrics`
- Intégration du service monitor de traefik dans prometheus (en gros collecte de la route metrics) : `curl http://<www.zzz>:9090/api/v1/label/__name__/values |jq|grep traefik

Si les métrique traefik sont exposées dans prometheus, vous pouvez maintenant les visualiser dans grafana.


# Visualisation
Pour accéder à grafana, vous devez auparavant ouvrir les accès :

-  Récupérer le mot de passe admin : `kubectl get secrets prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo`
-  Faire un port-forward sur le pod grafana `kubectl port-forward prometheus-grafana-654869cd-mn8qq 3000:3000`
-  Y accéder localement via firefox ou sur une autre machine avec un tunnel ssh : ssh -L 3000:localhost:3000 <monPoteIP>

Grafana, Kibana, sont les mêmes outils de visualisation de métriques et de surveillance d'alertes. Pour cette installation nous avons supprimé tous les `dashboards` de surveillance. Quand vous allez vous connecter dans Grafana vous ne verrez pas grand chose. 
- Vous pouvez explorer les metriques collectées : menu->explore, sélectionner une metrique et faites `run query`. Normalement la magie opère. Vous pouvez choisir des metriques de mesure de cpu par exemple.

Si vous commencez à voir quelque chose, vous pouvez installer un dashboard, ou en créer un, si vous êtes aventurier ou explorateur. 
Nous avons déposé sur la clé un dashboard json `dashboard-grafana-17346_rev9.json` permettant de visualiser les  métriques traefik.


Normalement vous pouvez consulter le dashboard traefik qui est... toujours vide, a part la zone indiquant les instances traefik qui devrait être à 1.

# Et si on testait la charge ?
Sur votre machine ou sur une autre, vous pouvez tester : 

ab -n 10000 -c 300 -H "Host: minecraft.localhost" "http://<monPoteIp>/display_skin?username=toto"

N'oubliez pas de recharger la page grafana, pour voir les métrique se mettre à jour plus rapidement. 


## Voilà la mise en place d'une infrastructure de supervision est terminée... Ou ce n'est que le début.


  - La commande `kubectl get servicemonitors.monitoring.coreos.com`. Trouve bien un servicemonitor pour traefik.
      ```
      NAME      AGE
      traefik   48s
      ```
  - La commande `kubectl describe servicemonitors.monitoring.coreos.com`. Affiche bien un label `release=prometheus`.
      ```
      Name:         traefik
      Namespace:    default
      Labels:       app.kubernetes.io/component=metrics
              app.kubernetes.io/instance=traefik-default
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=traefik
              helm.sh/chart=traefik-39.0.5
              release=prometheus   <--- ICI
      Annotations:  meta.helm.sh/release-name: traefik
      ...
      ```   

