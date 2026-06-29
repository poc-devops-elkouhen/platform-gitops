# Spec fonctionnelle — platform-gitops

## Modèle de synchronisation ArgoCD

ArgoCD surveille ce dépôt via l'Application racine (`root`) appliquée une
seule fois à la main depuis `platform-cicd`. Cette Application cible
`argocd/managed/` sur la branche `main`.

Tout ce qui se trouve dans `argocd/managed/` est donc géré par ArgoCD :
modifications committées → réconciliation automatique sur le cluster.

## Inventaire des applications

Les applications sont déclarées dans `argocd/apps/*.yaml`. Format minimal :

```yaml
name: myapp
services:
  - myapp-svc
  - myapp-gui
manifests:
  path: k8s
```

Les champs non déclarés sont dérivés par convention dans `platform_inventory.py` :

| Champ omis | Valeur dérivée |
|------------|----------------|
| `manifests.projectPath` | `root/<name>-iac` |
| `manifests.argocdRepoURL` | URL in-cluster GitLab |
| `code.projectPath` | `root/<name>` |
| `environments` | `dev`, `rec`, `preprod` (si `hasPreprod: true`), `prod` |
| `showcaseService` | premier service dont le nom se termine par `gui/ui/web/front` |

## Composants plateforme (`argocd/platform/`)

| Dossier | Contenu |
|---------|---------|
| `argocd-config/` | `argocd-cm.yaml` (Dex OIDC), `argocd-rbac-cm.yaml`, `argocd-cmd-params-cm.yaml` |
| `argocd-ui/` | HTTPRoute ArgoCD, HTTPRoute Dex, Service NodePort Dex |
| `gitlab-routes/` | HTTPRoutes GitLab (UI, registry, SSH) |
| `gitlab-minio-patch/` | Patch Kustomize pour les buckets Minio GitLab |
| `registry/` | Déploiement, Service et HTTPRoute du registry Docker |

## Flux d'onboarding d'une application

1. Créer `argocd/apps/<app>.yaml` (format minimal ci-dessus).
2. Depuis `platform-cicd` : `make argocd-apps-render` → commit `argocd/managed/apps-appset.yaml`.
3. Pousser sur `main` → ArgoCD détecte le changement → crée les Applications.
4. Depuis `toolbox` : `make gitlab-seed` → crée les projets GitLab + seed + CI.

## Promotion des applications

Les branches d'environnement (`dev`, `rec`, `preprod`) du dépôt manifests
(`helloworld-iac`) sont mises à jour par la CI applicative via `deploy.py`.
Ce dépôt ne porte pas de logique de promotion — il porte uniquement la
déclaration de quelles branches ArgoCD doit surveiller.
