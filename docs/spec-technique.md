# Spec technique — platform-gitops

## Structure du dépôt

```
argocd/
  apps.yaml                    Métadonnées globales (domaine, registry, repoURL GitOps)
  apps/                        Inventaire des applications (un fichier par app)
    helloworld.yaml
  managed/                     GÉNÉRÉ — ne pas éditer à la main
    apps-appset.yaml           AppProject + ApplicationSet par app (généré par render-argocd-apps.py)
    gitlab.yaml                Application ArgoCD pour GitLab (chart Helm)
    platform-appset.yaml       ApplicationSet pour les composants plateforme
    gitlab-iac-repo-creds.yaml Secret ArgoCD pour accéder aux dépôts manifests GitLab
  platform/                    Manifests des composants plateforme
    argocd-config/             Kustomization : argocd-cm, argocd-rbac-cm, argocd-cmd-params-cm
    argocd-ui/                 HTTPRoutes ArgoCD et Dex
    gitlab-routes/             HTTPRoutes GitLab
    gitlab-minio-patch/        Patch Kustomize Minio
    registry/                  Déploiement registry:2 + Service + HTTPRoute
docs/
  prd.md
  spec-fonctionnelle.md
  spec-technique.md
```

## `argocd/apps.yaml` — métadonnées plateforme

Ce fichier fixe les constantes résolues par `platform_inventory.py` :

```yaml
platform:
  domain: 192.168.33.100.nip.io
  repoURL: https://github.com/poc-devops-elkouhen/platform-gitops
  targetRevision: main
  registry:
    host: registry.registry.svc.cluster.local:5000

gitlab:
  internalHost: gitlab-webservice-default.gitlab.svc.cluster.local:8181

appsDir: apps
```

`repoURL` est l'URL externe (GitHub) utilisée pour le bootstrapping initial
d'ArgoCD. Une fois GitLab opérationnel, ArgoCD peut être reconfiguré pour
pointer vers l'instance GitLab interne.

## `argocd/managed/apps-appset.yaml` — format généré

Contient, dans l'ordre :
1. Un `AppProject` par app (whitelist `Namespace` pour `CreateNamespace=true`).
2. Un `ApplicationSet` avec un générateur `list` et un template Go.

Le template génère des `Application` nommées `<app>-<env>` dans le namespace
`argocd`, pointant vers la branche d'environnement du dépôt manifests de l'app.

## Configuration ArgoCD (`argocd/platform/argocd-config/`)

- **`argocd-cm.yaml`** : configure Dex avec le provider GitLab OIDC.
  Les valeurs `dex.gitlab.clientID` et `dex.gitlab.clientSecret` sont
  injectées dans `argocd-secret` par `gitlab-dex-oauth-app.py` lors du bootstrap.
- **`argocd-rbac-cm.yaml`** : mappe les groupes GitLab sur les rôles ArgoCD
  (`role:admin` pour tous les utilisateurs connectés dans ce POC mono-équipe).
- **`argocd-cmd-params-cm.yaml`** : active `server.insecure=true` pour exposer
  ArgoCD derrière Traefik sans TLS terminé par ArgoCD.

## Registry (`argocd/platform/registry/`)

Déploiement d'un registry `registry:2` (Docker Distribution) sans
authentification, exposé :
- **In-cluster** via un `Service ClusterIP` sur le port 5000.
- **Externe** via une `HTTPRoute` Traefik sur `registry.<domaine>`.

Pas de persistance au-delà du `PersistentVolumeClaim` local-path-provisioner —
les images sont perdues en cas de destruction du PVC.
