# SRE Lab - GitOps

Ce dépôt est la **Source de Vérité** (Source of Truth) pour l'état du cluster Kubernetes.
Il est surveillé par **ArgoCD** qui synchronise automatiquement le cluster avec le contenu de ce dépôt.

## Structure

*   **apps/** : Applications métiers (ex: Guestbook, Démos...).
*   **infrastructure/** : Composants d'infrastructure (ex: Prometheus, Cert-Manager, NFS...).
*   **bootstrap/** : Configuration initiale d'ArgoCD (App of Apps).

## Workflow

1.  On définit une application ici (YAML ou Helm).
2.  On commit & push.
3.  ArgoCD détecte le changement et l'applique sur le cluster.
