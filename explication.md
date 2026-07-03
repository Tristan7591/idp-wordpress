# Évolution du dépôt `idp-wordpress`

## Objectif

Ce dépôt construit une petite Internal Developer Platform (IDP) pour WordPress. Le développeur demande des capacités de plateforme avec des ressources Kubernetes simples (`Database` et `Environment`) ; Crossplane traduit ensuite ces demandes en releases Helm, tandis qu'Argo CD synchronise l'état déclaré dans Git avec le cluster.

Le dépôt personnel utilisé par la plateforme est :

```text
https://github.com/Tristan7591/idp-wordpress.git
```

## Architecture actuelle

```text
Développeur
    │ Pull Request : Database / Environment
    ▼
GitHub : Tristan7591/idp-wordpress
    │
    ▼
Argo CD
    ├── tp-perso-crossplane-bootstrap
    │      └── provider-helm + runtime + RBAC
    ├── tp-perso-crossplane-resources
    │      └── ProviderConfig + XRD + Compositions
    └── tp-perso-wordpress
           └── demandes Database + Environment
                  │
                  ▼
              Crossplane
                  ├── MariaDB sur NFS
                  ├── Memcached
                  └── vCluster puis WordPress (encore à finaliser)
```

La séparation en trois applications évite de créer une ressource avant que son API ou son provider existe.

## Ce qui manquait dans le dépôt de base

### Provider Helm absent

Les compositions créent des ressources `Release.helm.crossplane.io` et `ProviderConfig.helm.crossplane.io`, mais aucun `provider-helm` n'était déclaré. Les CRD correspondantes n'existaient donc pas.

Symptômes :

```text
CustomResourceDefinition "Release.helm.crossplane.io" not found
CustomResourceDefinition "ProviderConfig.helm.crossplane.io" not found
```

### Mauvais ordre d'installation

Les `Composition` étaient appliquées avant les XRD. Une composition est un modèle ; elle référence un type composite qui doit déjà être déclaré par une `CompositeResourceDefinition`.

Ordre correct :

```text
Crossplane
→ providers et permissions
→ ProviderConfig
→ XRD
→ Compositions
→ claims/XR
```

### ProviderConfig `default` absent

Les releases MariaDB, Memcached et vCluster utilisent toutes :

```yaml
providerConfigRef:
  name: default
```

Ce `ProviderConfig` n'existait pas. Il a été ajouté avec `InjectedIdentity` afin que le provider Helm utilise son ServiceAccount dans le cluster local.

### Permissions insuffisantes

Le ServiceAccount généré initialement pour `provider-helm` ne pouvait pas créer de `Deployment` :

```text
kubectl auth can-i create deployments ...
no
```

Un `DeploymentRuntimeConfig` crée maintenant un ServiceAccount stable nommé `provider-helm`. Un `ClusterRoleBinding` lui donne `cluster-admin`.

Ce droit large est acceptable uniquement pour ce laboratoire Vagrant, car le provider installe notamment vCluster et des ressources cluster-scoped. Il doit être remplacé par des permissions minimales dans un environnement partagé ou de production.

### Argo CD absent et manifests incompatibles

Argo CD n'était pas installé. Les `Application` étaient également placées dans le namespace `argo`, alors que l'installation standard utilise `argocd`.

Le dépôt utilisait aussi :

```yaml
automated: false
```

Sur la CRD Argo CD actuelle, `automated` est un objet et non un booléen. Pour conserver une synchronisation manuelle, le champ doit être omis.

### Dépôt Git incorrect

Les applications Argo CD lisaient encore le dépôt d'origine. Les `repoURL` pointent maintenant vers le dépôt personnel de Tristan.

### Images Bitnami versionnées indisponibles

Les anciens charts Bitnami demandaient notamment :

```text
docker.io/bitnami/memcached:1.6.17-debian-11-r35
docker.io/bitnami/mariadb:10.6.12-debian-11-r16
```

Ces tags ne sont plus disponibles dans le catalogue public principal. Le résultat était `ImagePullBackOff`.

Changer uniquement `image.repository` vers l'image Docker officielle n'est pas sûr : les templates Bitnami attendent leurs scripts, chemins et conventions internes. Les compositions backend ont donc été migrées vers des charts HelmForge open source qui utilisent les images Docker officielles gratuites :

```text
docker.io/library/memcached:1.6.42
docker.io/library/mariadb:12.3.2
```

Les versions de charts et d'images sont épinglées pour rendre les déploiements reproductibles.

### Stockage Kubernetes absent

Un disque était disponible sur la VM, et les clients NFS étaient installés, mais il manquait :

- un serveur NFS actif ;
- un export NFS ;
- un provisioner dynamique Kubernetes ;
- une `StorageClass` par défaut.

Le PVC MariaDB restait donc `Pending` avec :

```text
pod has unbound immediate PersistentVolumeClaims
```

Le serveur NFS de `master` exporte désormais `/srv/nfs/kubernetes` vers `192.168.99.0/24`. `nfs-subdir-external-provisioner` fournit la StorageClass `nfs-client`, avec une politique `Retain`.

## Modifications versionnées

| Fichier | Modification | Pourquoi |
|---|---|---|
| `argo-cd/argo-manifests/Application-bootstrap.yml` | Nouvelle application de bootstrap | Installer le provider avant les API qui en dépendent |
| `argo-cd/argo-manifests/Application.yml` | Namespace `argocd`, dépôt personnel, suppression de `automated: false` | Compatibilité Argo CD et bonne source Git |
| `argo-cd/crossplane-bootstrap/provider-helm.yml` | Provider Helm `v1.2.0` épinglé | Créer et gérer les `Release` Helm |
| `argo-cd/crossplane-bootstrap/provider-helm-runtime.yml` | ServiceAccount stable et RBAC | Permettre au provider d'agir dans le cluster de démo |
| `resources/providerconfig-helm-default.yml` | `ProviderConfig/default` avec `InjectedIdentity` | Connecter le provider au cluster principal |
| `resources/composition-database-memcached.yml` | HelmForge `1.0.4`, image officielle Memcached `1.6.42` | Remplacer l'image Bitnami introuvable |
| `resources/composition-database-mariadb.yml` | HelmForge `2.0.3`, image officielle MariaDB `12.3.2`, StorageClass `nfs-client` | Image gratuite maintenue et stockage persistant |
| `resources/composition-env-x-wordpress.yml` | Mise à jour de vCluster et ajustements du chart WordPress | Préparer l'environnement applicatif ; cette partie reste à finaliser |

Commits principaux :

```text
d6f15fa Configure personal GitOps platform with Crossplane and Argo CD
bd9272d Replace Bitnami backends with free official images
```

## Modifications faites sur le cluster, non encore versionnées

Les éléments suivants existent sur la VM mais ne sont pas encore entièrement reconstruits par GitOps :

- installation de Crossplane `v1.15.0` ;
- installation d'Argo CD dans `argocd` ;
- installation et configuration du serveur NFS sur `master` ;
- export `/srv/nfs/kubernetes` ;
- installation Helm de `nfs-subdir-external-provisioner` ;
- création de la StorageClass `nfs-client` par défaut.

Une reconstruction complète du cluster exige donc encore des scripts ou applications Argo CD dédiés à Crossplane, Argo CD et NFS.

## État validé

Les composants suivants ont été testés :

```text
provider-helm       INSTALLED=True, HEALTHY=True
tp-perso-db         SYNCED=True, READY=True
tp-perso-cache      SYNCED=True, READY=True
MariaDB             Running/Ready
Memcached           Running/Ready
PVC MariaDB         Bound sur nfs-client
```

Tests fonctionnels réalisés :

- création, lecture et suppression d'une table temporaire MariaDB ;
- écriture et lecture d'une clé Memcached ;
- contrôle de la création du répertoire MariaDB sur l'export NFS.

## Limites actuelles

### WordPress n'est pas encore finalisé

La composition d'environnement contient encore un ancien chart WordPress Bitnami. Il faut le migrer vers une image officielle gratuite et réimplémenter proprement l'intégration Memcached avant de synchroniser l'application WordPress.

### Secrets externes au dépôt

Les mots de passe initialement présents dans les compositions ont été retirés. MariaDB référence désormais le Secret Kubernetes `tp-perso-db-credentials`. La composition WordPress référence `wordpress-db-credentials`, à provisionner dans le vCluster avant le déploiement du chart. Ces Secrets ne sont pas versionnés ; la cible reste de les alimenter avec External Secrets/Vault plutôt que manuellement.

Un `.gitignore` exclut aussi les fichiers `.env`, kubeconfigs, clés privées, certificats, fichiers de credentials locaux et états d'outils. `argo-cd/argo-manifests/Repository.yml` a été retiré de l'index Git : le dépôt étant public, Argo CD peut le lire sans Secret de dépôt. Le fichier local reste disponible sur la VM mais ne sera plus poussé.

### RBAC trop large

`provider-helm` possède `cluster-admin`. Une plateforme partagée doit utiliser des rôles dédiés, des namespaces isolés et éventuellement plusieurs ProviderConfig.

### Contrat développeur trop pauvre

Les XRD exposent très peu de paramètres. Le stockage, les versions, les hostnames et plusieurs identifiants sont codés dans les compositions. Ils doivent devenir soit des paramètres encadrés, soit des choix imposés par classe de service.

### Synchronisation encore manuelle

Les applications n'activent pas `syncPolicy.automated`. C'est utile pendant la mise au point, mais un véritable flux GitOps doit automatiser au moins `selfHeal`, puis éventuellement `prune` après validation.

## Parcours développeur actuel

Le développeur ne doit pas modifier les charts, les providers ou les compositions. Il consomme le contrat de plateforme en créant des claims/XR.

### Demander les backends

```yaml
apiVersion: tp-perso.io/v1alpha1
kind: Database
metadata:
  name: equipe-blog-db
spec:
  compositionSelector:
    matchLabels:
      provider: local
      type: dev
      kind: sql
---
apiVersion: tp-perso.io/v1alpha1
kind: Database
metadata:
  name: equipe-blog-cache
spec:
  compositionSelector:
    matchLabels:
      provider: local
      type: dev
      kind: cache
```

Le nom du XR devient le nom de la release. Les services obtenus suivent actuellement la convention :

```text
equipe-blog-db-mariadb
equipe-blog-cache-memcached
```

### Demander un environnement

```yaml
apiVersion: tp-perso.io/v1alpha1
kind: Environment
metadata:
  name: equipe-blog
  namespace: tp-perso
spec:
  writeConnectionSecretToRef:
    name: equipe-blog
  compositionSelector:
    matchLabels:
      type: development
  parameters:
    installInfra: false
```

La promesse cible est : un claim `Environment` crée un vCluster, configure un provider pour ce vCluster et y installe WordPress. Cette chaîne n'est pas encore prête pour une utilisation développeur autonome.

### Flux GitOps recommandé

1. Créer une branche depuis `main`.
2. Ajouter les claims dans un dossier applicatif, idéalement `environments/<équipe>/<application>/`.
3. Ouvrir une Pull Request.
4. Laisser la CI valider YAML, schémas et politiques.
5. Fusionner après revue.
6. Argo CD synchronise l'application concernée.
7. Le développeur observe le statut des claims, sans administrer directement Helm.

Exemple :

```bash
git checkout -b app/equipe-blog
mkdir -p environments/equipe-blog/wordpress

# Ajouter les claims dans ce dossier.

git add environments/equipe-blog/wordpress
git commit -m "Request WordPress environment for equipe-blog"
git push -u origin app/equipe-blog
```

Dans l'organisation actuelle du dépôt, Argo CD surveille plutôt les trois fichiers `tp-perso-*.yml` à la racine de `crossplane-resources-manifests`. Le dossier `environments/` est une évolution recommandée pour éviter de mélanger les demandes développeur et les définitions de plateforme.

### Observer le déploiement

```bash
kubectl get applications -n argocd
kubectl get database
kubectl get environment -A
kubectl get releases.helm.crossplane.io
kubectl get pods,pvc,svc -n tp-perso
```

Le développeur doit principalement consulter ses claims et les champs `SYNCED`/`READY`. Les `Release`, providers et compositions restent des détails d'implémentation gérés par l'équipe plateforme.

## Construire autour de la plateforme

### Stabiliser le contrat XRD

Le contrat développeur devrait exposer seulement les paramètres réellement utiles :

```yaml
spec:
  parameters:
    size: small
    storageGiB: 5
    environment: dev
    hostname: blog.example.internal
    databaseClass: mariadb-dev
    cacheClass: memcached-dev
```

Les versions d'images, chart repositories, règles de sécurité et classes de stockage doivent rester sous contrôle de l'équipe plateforme.

### Proposer plusieurs classes de service

Les labels de composition permettent d'ajouter des implémentations sans changer l'API développeur :

```text
type=dev, provider=local, kind=sql
type=prod, provider=cloud, kind=sql
type=dev, provider=local, kind=cache
```

Un même claim peut ainsi sélectionner une base locale pour le développement ou une base managée pour la production.

### Isoler les responsabilités Git

Structure cible :

```text
platform/
  bootstrap/       # providers, fonctions, RBAC
  definitions/     # XRD
  compositions/    # implémentations
  policies/        # Kyverno/Gatekeeper
environments/
  team-a/blog/     # claims développeur
  team-b/shop/
```

### Gérer les secrets

- ne jamais placer les mots de passe dans les compositions ou les claims ;
- référencer des Secrets existants ;
- utiliser External Secrets avec Vault, SOPS ou un gestionnaire cloud ;
- appliquer la rotation et limiter les droits par namespace.

### Ajouter des garde-fous

- validation des XRD par schéma OpenAPI ;
- quotas CPU, mémoire et stockage ;
- LimitRange et ResourceQuota par équipe ;
- NetworkPolicy ;
- politiques d'images autorisées et signatures ;
- sauvegarde et restauration testées pour MariaDB.

### Ajouter l'observabilité

- métriques Crossplane et Argo CD ;
- métriques MariaDB/Memcached ;
- tableaux de bord par environnement ;
- événements et conditions remontés au développeur ;
- alertes sur `Ready=False`, PVC presque plein et échec de synchronisation.

### Ajouter une interface développeur

Une fois le contrat stabilisé, Backstage ou un formulaire interne peut générer les claims et ouvrir automatiquement une Pull Request. L'interface ne doit pas appeler Helm directement : Git reste la source de vérité et Crossplane reste le moteur de provisioning.

## Vérification de bout en bout

```bash
# GitOps
kubectl get applications -n argocd

# Control plane
kubectl get providers.pkg.crossplane.io
kubectl get xrd
kubectl get compositions

# Demandes développeur
kubectl get database
kubectl get environment -A

# Ressources concrètes
kubectl get releases.helm.crossplane.io
kubectl get pods,pvc,svc,ingress -n tp-perso
```

Test MariaDB :

```bash
DB_PASSWORD=$(kubectl get secret tp-perso-db-credentials -n tp-perso \
  -o jsonpath='{.data.mariadb-user-password}' | base64 -d)

kubectl exec -n tp-perso tp-perso-db-mariadb-0 -- \
  env MARIADB_PWD="$DB_PASSWORD" \
  mariadb -ubn_wordpress bitnami_wordpress \
  -e 'SELECT 1 AS ok;'

unset DB_PASSWORD
```

Test Memcached :

```bash
kubectl run memcached-test -n tp-perso --rm -i --restart=Never \
  --image=busybox:1.37.0 -- \
  sh -c 'printf "set health 0 60 2\r\nok\r\nget health\r\nquit\r\n" | nc tp-perso-cache-memcached 11211'
```

## Prochaines étapes prioritaires

1. Migrer WordPress vers une image officielle gratuite et restaurer l'intégration cache.
2. Corriger les différences Argo CD persistantes sur les `Composition`.
3. Déclarer l'installation NFS et les prérequis du cluster dans Git.
4. Alimenter les Secrets externes avec Vault/External Secrets et organiser leur rotation.
5. Réduire les permissions de `provider-helm`.
6. Ajouter des paramètres utiles aux XRD et valider les claims en CI.
7. Créer un dossier `environments/` et une application Argo CD par environnement ou équipe.
8. Activer progressivement `selfHeal`, puis `prune` après tests de suppression.
