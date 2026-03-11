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

`curl http://localhost:9090/api/v1/label/__name__/values

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






Après avoir démarré votre machine, vous pouvez installer la nouvelle version de traefik fourni sur la clé. L'installation est modifiée, car pour que trafik puisse être surveillé par **prometheus** il faut qu'il déclare un objet **servicemonitor** dont la déclaration est fournie par prometheus... Nous avons donc un soucis d'œuf et de poule.   De plus nous souhaitons installer des versions minimales de nos infrastructures. Enfin, le **servicemonitor** installé par traefik respecte bien la spécification de prometheus, mais se déclare mal... En effet il devrait déclarer un **label** spécifique qui n'est pas positionné.   
Le script `/home/user/startTraefik.sh` corrige ces deux soucis. 

1. Lancer le script
2. Vérifier les éléments suivants : 
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

Si tout est ok, traefik met à disposition une route metrics que vous pouvez consulter localement et à distance. Pour consulter localement, vous pouvez ouvrir le port de la manière suivante, en adaptant les numéros du pod. 
`kubectl port-forward traefik-<xxx>-<yyy> 9100:9100` 

Vous pouvez tester votre accès dans une autre fenêtre avec `curl localhost:9100/metrics`. Vous devez voir des 'traefikxxxx' dans les dernières metrics.

:question: Savez-vous rendre disponible cette accès aux machines externes ?


Si tout se passe bien, prometheus sais maitenant détecter automatiquement votre fournisseur de metriques. Car il déclare un service monitor contenant un label `release=prometheus`. Il reste donc à installer prometheus et vérifier cela. 







1. Prérequis : 10GB$
2. Réseau : ./startnet.sh
3. Démarrer un serveur k3s : ./startk3sServer.sh
4. Passer admin : sudo su -
5. Toujours faire un : export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
6. Lancer k9s : k9s
 . ./startTraefik.sh
   

