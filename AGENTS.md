# AGENTS.md — platform-gitops

## Rôle du dépôt

`platform-gitops` est la source de vérité GitOps du POC. ArgoCD surveille ce
dépôt en continu via l'Application racine (`argocd/root-app.yaml` dans
`platform-cicd`). Tout changement committé ici est réconcilié automatiquement
sur le cluster.

## Structure

```
argocd/
  apps.yaml              Métadonnées globales de la plateforme (domaine, registry)
  apps/                  Inventaire des applications (un fichier YAML par app)
  managed/               Fichiers GÉNÉRÉS — ne pas éditer à la main
    apps-appset.yaml     ApplicationSet ArgoCD généré par render-argocd-apps.py
    gitlab.yaml          Application ArgoCD pour GitLab
    platform-appset.yaml ApplicationSet pour les composants plateforme
    gitlab-iac-repo-creds.yaml Secret ArgoCD pour le dépôt manifests GitLab
  platform/              Manifests des composants plateforme
    argocd-config/       ConfigMaps ArgoCD (RBAC, SSO Dex, paramètres)
    argocd-ui/           Routes HTTP ArgoCD et Dex
    gitlab-routes/       Routes HTTP GitLab
    gitlab-minio-patch/  Patch Minio pour GitLab
    registry/            Déploiement du registry Docker interne
```

## Règles critiques

- **`argocd/managed/` est entièrement généré** par `scripts/render-argocd-apps.py`
  dans `platform-cicd`. Ne jamais éditer ces fichiers à la main — les
  modifications seraient écrasées à la prochaine exécution de `make argocd-apps-render`.
- **`argocd/apps/*.yaml`** sont les seuls fichiers à éditer pour ajouter ou
  modifier une application. Format minimal requis : `name`, `services`, `manifests.path`.
- **`argocd/apps.yaml`** contient les constantes de plateforme (domaine, registry,
  URL GitOps). Modifier ici impacte tous les dérivés.

## Ajouter une application

1. Créer `argocd/apps/<app>.yaml` avec au minimum :
   ```yaml
   name: <app>
   services:
     - <service-1>
     - <service-2>
   manifests:
     path: k8s
   ```
2. Depuis `platform-cicd` : `make argocd-apps-render` puis committer `apps-appset.yaml`.
3. Depuis `toolbox` : `make gitlab-seed` pour créer les projets GitLab.

## Ce qu'il ne faut pas faire

- Ne pas éditer `argocd/managed/` directement.
- Ne pas committer de secrets non chiffrés dans ce dépôt.
- Ne pas pousser sur `main` sans avoir vérifié que `make check-generated` passe
  dans `platform-cicd`.
