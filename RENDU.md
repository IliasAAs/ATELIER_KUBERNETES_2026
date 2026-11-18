# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** ASRI Ilias
**Groupe :** RS

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > 3 nœuds : `k3d-lab-server-0` (rôle `control-plane`, le « master » : API Server, scheduler, stockage de l'état du cluster, contrôleurs) et `k3d-lab-agent-0` / `k3d-lab-agent-1` (workers : ils exécutent les pods des applications).
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` liste des conteneurs Docker : un par nœud (`k3d-lab-server-0`, `k3d-lab-agent-0`, `k3d-lab-agent-1`, image `rancher/k3s`) plus un conteneur load balancer (`k3d-lab-serverlb`) qui relie le port 8081 du Codespace au port 80 du cluster. Un « nœud » k3d est donc un conteneur Docker qui fait tourner k3s (Kubernetes léger) : les pods tournent dans des conteneurs à l'intérieur de ces conteneurs-nœuds.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > Un **Deployment** `web`, qui a créé un **ReplicaSet** (`web-<hash>`), qui a lui-même créé un **Pod** (`web-<hash>-<id>`). `kubectl expose` a ajouté un **Service** `web` (ClusterIP). Le Service `kubernetes` du namespace `default` existait déjà.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Un nouveau pod est apparu immédiatement (`Terminating` pour l'ancien, puis un nouveau `ContainerCreating` → `Running`), avec un **nom différent** (nouveau suffixe aléatoire). C'est la boucle de réconciliation : le ReplicaSet compare l'état désiré (1 réplica) à l'état réel (0 pod) et recrée un pod.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non. Le système de fichiers d'un conteneur est éphémère : le nouveau pod repart de l'image `nginx` d'origine (page « Welcome to nginx! »). Une modification faite à la main avec `kubectl exec` est perdue au redémarrage du pod.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Un `kubectl port-forward svc/web` s'attache en réalité à **un seul pod** (celui choisi au lancement), pas au Service dans son ensemble. Quand ce pod est supprimé, le tunnel est coupé (visible dans `/tmp/web.log`) et il faut le relancer vers le nouveau pod.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `create` est **impératif** : il crée une ressource et échoue (`AlreadyExists`) si elle existe déjà. `apply` est **déclaratif** : on fournit l'état désiré (fichier YAML) et Kubernetes crée la ressource si elle n'existe pas ou la met à jour si elle diffère ; la commande est idempotente et le fichier peut être versionné dans Git.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` désigne, par labels, les pods vers lesquels le Service répartit le trafic. S'il ne correspond à aucun pod, le Service n'a **aucun endpoint** (`kubectl get endpoints` vide) : les requêtes ne sont servies par personne (connexion refusée / timeout, ou erreur 503 via un Ingress), alors que le Service lui-même existe.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
   NAME                    READY   STATUS    RESTARTS   AGE    IP          NODE               NOMINATED NODE   READINESS GATES
   site-54b5858c4c-d9nrd   1/1     Running   0          6s     10.42.0.6   k3d-lab-agent-0    <none>           <none>
   site-54b5858c4c-w4x6n   1/1     Running   0          2m9s   10.42.2.6   k3d-lab-server-0   <none>           <none>
   site-54b5858c4c-xwf4g   1/1     Running   0          6s     10.42.1.8   k3d-lab-agent-1    <none>           <none>
   ```
   Les 3 réplicas sont répartis sur les 3 nœuds du cluster (1 pod par nœud) : le scheduler a placé un pod sur chacun, y compris sur le control-plane, qui accepte les applications dans k3s.
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Le nouveau pod était en `ErrImagePull` puis `ImagePullBackOff` : l'image `nginx:9.99` n'existe pas. La cause exacte est donnée par `kubectl describe pod -l app=site`, section *Events* (échec du pull de l'image `nginx:9.99`, qui n'existe pas).
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > Oui. Le Deployment fait un *rolling update* : il ne retire les anciens pods qu'au fur et à mesure que les nouveaux passent `Ready`. Le nouveau pod ne devenant jamais prêt, les anciens pods (version précédente) ont continué à servir le trafic, sans coupure ; `kubectl rollout undo` a ensuite restauré la révision précédente.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://jubilant-space-engine-7vpwpj5p5v952vg9-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > Oui : `kubectl get deploy mario` retrouve le Deployment créé depuis Portainer. Les deux outils sont des clients de la même **API Server** Kubernetes (sur le nœud control-plane) : `kubectl` s'y connecte avec le kubeconfig, Portainer avec le ServiceAccount du pod qui tourne dans le cluster. L'état est le même, quel que soit l'outil.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > A (Ingress Traefik, site accessible sur le port 8081) et B (readinessProbe HTTP sur / et requests/limits CPU et mémoire) réalisés.

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
