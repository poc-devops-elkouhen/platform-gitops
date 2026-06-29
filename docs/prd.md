# PRD — platform-gitops

## Intention du projet

`platform-gitops` est la source de vérité GitOps du POC. Il centralise l'état
déclaratif de tous les composants plateforme (GitLab, ArgoCD, registry) et de
toutes les applications (`helloworld`, et toute future app onboardée).

La vision globale de la chaîne CI/CD est dans
`../../platform-cicd/docs/prd.md`.

## Produit attendu

Le dépôt doit fournir :

- un inventaire déclaratif des applications (`argocd/apps/*.yaml`) ;
- les Applications et ApplicationSets ArgoCD pour tous les composants et toutes
  les apps ;
- les manifests des composants plateforme (routes, config ArgoCD, registry) ;
- un mécanisme de génération automatique de l'ApplicationSet applicatif.

## Utilisateurs cibles

- **ArgoCD** (consommateur principal) : synchronise ce dépôt en continu depuis
  l'Application racine.
- **Mainteneur plateforme** : ajoute ou retire des applications en éditant
  `argocd/apps/*.yaml`.
- **Scripts `platform-cicd`** : génèrent `argocd/managed/apps-appset.yaml` via
  `render-argocd-apps.py`.

## Critères d'acceptation

- Toute modification commitée sur `main` est réconciliée par ArgoCD sans
  intervention manuelle.
- Ajouter un fichier `argocd/apps/<app>.yaml` + régénérer `apps-appset.yaml`
  suffit à créer les Applications ArgoCD pour la nouvelle app.
- `argocd/managed/` est entièrement généré et ne contient aucune configuration
  manuelle.
- Les composants plateforme (GitLab, registry, ArgoCD config) sont déclarés dans
  `argocd/platform/` et synchronisés par ArgoCD.

## Non-objectifs

- Contenir des secrets non chiffrés.
- Décrire la logique CI/CD applicative (portée par `ci-templates`).
- Héberger les manifests applicatifs (portés par les dépôts `*-iac` dédiés).
