Nous allons installer une infrastructure de supervision minimale. Cette infrastructure surveille l'application Web minecraft qui passe par traefik pour l'accès externe.   
L'infrastructure minimale repose sur les composant et l'architecture suivante.  


- **Traefik** est le gestionnaire d'accès de type API Gateway. C'est le composant que nous voulons surveiller. 
  - Il expose des métriques via la route localhost:9100/metrics
- Cette route est régulièrement interrogée par le gestionnaire de sonde **Prometheus**.
  - Il peut être accédé via une route localhost:9090
- Les valeur collectées et aggrégées par prometheus peuvent être visualisées graphiquement par **graphana** 

metrics --> prometheus --> graphana, sont les éléments les plus traditionnels des infrastructures de supervision de système. Le nom des composants peuvent changer, mais les trois rôles clés : data -> aggregation -> visualisation existent toujours. 
Nous vous proposons la mise en place de cette infrastructure, puis la vérification via l'usage de l'application web minecraft.

# Installation de Traefik
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

# Installation de prometheus
Si tout se passe bien, prometheus sais maitenant détecter automatiquement votre fournisseur de metriques. Car il déclare un service monitor contenant un label `release=prometheus`. Il reste donc à installer prometheus et vérifier cela. 











1. Prérequis : 10GB$
2. Réseau : ./startnet.sh
3. Démarrer un serveur k3s : ./startk3sServer.sh
4. Passer admin : sudo su -
5. Toujours faire un : export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
6. Lancer k9s : k9s
 . ./startTraefik.sh
   

