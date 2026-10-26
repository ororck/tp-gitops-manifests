# tp-gitops-manifests

## Ce que c'est

La source de vérité du cluster AKS du TP. Ce dépôt décrit tout ce qui tourne dans le cluster, à part ArgoCD lui-même qui est installé par le dépôt `tp-gitops-infra`.

## Ce que ça fait

ArgoCD lit ce dépôt et applique son contenu dans le cluster, en continu.

- Il installe ingress-nginx, cert-manager, Loki, Alloy et Grafana
- Il déploie l'application de démonstration dans le namespace `demo`
- Il expose l'API Gateway et Grafana en HTTPS sur `api.20.82.52.99.nip.io` et `grafana.20.82.52.99.nip.io`
- Il répare seul toute modification faite directement dans le cluster

La branche `main` est protégée, toute modification passe par une pull request.

## Comment ça marche

```
bootstrap/root-app.yaml      Application racine, appliquée une seule fois à la main
applications/                une Application ArgoCD par composant
components/                  manifestes propres aux composants (émetteurs de certificats, source Loki)
apps/demo/                   Deployments, Services et Ingress de la démonstration
docs/captures/               preuves de fonctionnement
```

L'Application `root` surveille le dossier `applications/`. Chaque fichier de ce dossier est une Application, avec la synchronisation automatique, le `prune` et le `selfHeal` actifs. Ajouter un fichier crée un composant, le supprimer le retire.

Les charts Helm sont épinglés à une version exacte. Le flux de livraison est le suivant.

1. La CI du dépôt `tp-gitops-app` construit l'image et la pousse sur le registre avec le SHA court comme tag
2. Le tag est reporté à la main dans `apps/demo/` par une pull request
3. ArgoCD voit le changement et met à jour le cluster

Pour le démontrer, supprimer un Deployment du namespace `demo` avec `kubectl delete`. ArgoCD le recrée en quelques secondes, comme le montre `docs/captures/05-self-healing.txt`.

### Choix qui méritent une justification

**Motif app of apps.** Une seule Application est appliquée à la main, toutes les autres en découlent. Cela évite de créer des Applications depuis l'interface et garde chaque composant versionné, relu en pull request et réparable par ArgoCD. L'ajout d'un composant se résume à un fichier.

**Chaîne de certificats auto-signée.** Il n'y a pas de domaine acheté, donc Let's Encrypt n'est pas utilisable sur des noms `nip.io`. cert-manager crée un émetteur auto-signé, qui signe un certificat d'autorité racine, puis un émetteur de type CA s'appuie sur ce certificat pour signer les certificats des deux hôtes. Le trafic est chiffré, mais le navigateur affiche un avertissement tant que la CA n'est pas approuvée. Les certificats serveur sont demandés par les Ingress grâce à l'annotation `cert-manager.io/cluster-issuer`.

**Collecteur de logs.** Alloy a été retenu à la place de Promtail. Le chart `grafana/promtail` est marqué obsolète, alors qu'Alloy est le collecteur maintenu par Grafana. Alloy lit les logs des pods par l'API Kubernetes et ajoute les labels `namespace`, `pod`, `container` et `app`. Il tourne en un seul exemplaire, un DaemonSet dupliquerait les logs avec cette méthode. Pour la même raison de maintenance, Loki et Grafana viennent du dépôt de charts `grafana-community`, les charts `grafana/grafana` étant obsolètes et `grafana/loki` étant devenu le chart de la version Enterprise.

**Ressources réduites.** Le cluster n'a qu'un node de 2 vCPU. Tous les composants ont des `requests` modestes, Loki n'a pas de persistance et les composants optionnels sont coupés.

**Différences ignorées pour cert-manager.** Le cainjector et l'admission d'AKS modifient le webhook de cert-manager après son installation. Ces champs sont ignorés dans l'Application pour qu'elle reste `Synced`.

## Dépendances

- Un cluster AKS avec ArgoCD installé par `tp-gitops-infra`
- Le dépôt `tp-gitops-app` et son registre, qui fournissent l'image
- Les charts ingress-nginx 4.15.1, cert-manager v1.21.3, loki 20.0.1, alloy 1.14.3 et grafana 14.1.0
- Une adresse IP publique sur le LoadBalancer d'ingress-nginx, pour les noms `nip.io`
