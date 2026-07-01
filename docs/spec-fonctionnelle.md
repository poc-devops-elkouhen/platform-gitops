# Spec fonctionnelle — platform-gitops

## Modèle de synchronisation ArgoCD

ArgoCD surveille ce dépôt via l'Application racine (`root`) appliquée une
seule fois à la main depuis `platform-cicd`. Cette Application cible
`argocd/managed/` sur la branche `main`.

Tout ce qui se trouve dans `argocd/managed/` est donc géré par ArgoCD :
modifications committées → réconciliation automatique sur le cluster.

Au provisioning initial, aucune application n'est déclarée : `argocd/apps/` et
`argocd/generated/apps/` restent vides. Les applications sont ajoutées après la
plateforme, une fois les projets GitLab provisionnés par Terraform.

`argocd/managed/` n'est pas une séparation fonctionnelle. C'est une sortie
générée qui contient les objets ArgoCD d'amorçage. Il peut contenir un
ApplicationSet générique qui pointe vers `argocd/generated/apps/*`, mais les
détails d'une application viennent de sa description `argocd/apps/<app>/app.yaml`.

## Composants plateforme (`argocd/platform/`)

| Dossier | Contenu |
|---------|---------|
| `argocd-config/` | `argocd-cm.yaml` (Dex OIDC), `argocd-rbac-cm.yaml`, `argocd-cmd-params-cm.yaml` |
| `argocd-ui/` | HTTPRoute ArgoCD, HTTPRoute Dex, Service NodePort Dex |
| `gitlab-routes/` | HTTPRoutes GitLab (UI, registry, SSH) |
| `gitlab-minio-patch/` | Patch Kustomize pour les buckets Minio GitLab |
| `registry/` | Déploiement, Service et HTTPRoute du registry Docker |

## Flux d'onboarding d'une application

L'onboarding applicatif ajoute un dossier `argocd/apps/<app>/` contenant
`app.yaml`, la description source de l'application : modules/services,
environnements optionnels et dépôts code/IaC. Les AppProject, ApplicationSet par
environnement et éventuels credentials repo sont générés sous
`argocd/generated/apps/<app>/`.

La création des projets GitLab et des dépôts `*-iac` n'est plus portée par
`toolbox`. Elle est déclarée dans le dépôt `gitlab-projects-iac` et appliquée
par le `Terraform/gitlab-iac` exécuté par tf-controller.

## Promotion des applications

Les branches d'environnement (`dev`, `rec`, `preprod`) du dépôt manifests
(`*-iac`) sont mises à jour par la CI applicative via `deploy.py`. Ce dépôt ne
porte pas la logique de promotion ; il déclare seulement, via `app.yaml`, les
branches applicatives suivies par ArgoCD. Le contenu des dépôts applicatifs et
les variables nécessaires aux pipelines doivent être initialisés par les jobs
CI/CD applicatifs prévus pour cela.
