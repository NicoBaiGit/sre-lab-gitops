# Infrastructure

Ce dossier contient les composants techniques du cluster, gérés par ArgoCD.

## Composants

### 1. NFS Provisioner (`nfs-provisioner.yaml`)

Permet au cluster de créer dynamiquement des volumes persistants (PVC) sur le NAS Synology.

**Configuration requise sur le NAS (Synology DSM) :**

1.  **Activer NFS** : Panneau de config > Services de fichiers > NFS > Activer (NFSv3 ou v4.1).
2.  **Créer un dossier partagé** :
    *   Nom : `k8s-data`
    *   Emplacement : `/volume1/k8s-data` (à adapter selon votre volume).
3.  **Permissions NFS** (Onglet Autorisations NFS du dossier) :
    *   **IP Client** : `192.168.1.120` (IP du serveur K3s) ou `192.168.1.0/24`.
    *   **Privilège** : Lecture/Écriture.
    *   **Squash** : `Mappage de tous les utilisateurs sur admin` (all_squash).
        *   *Pourquoi ?* Pour éviter les problèmes de permissions ("Permission Denied") quand un conteneur tourne avec un UID non-root (ex: Postgres, Redis).
    *   **Sécurité** : `sys`.
    *   **Ports non privilégiés** : ✅ **Cocher "Permettre les connexions à partir des ports non privilégiés (>1024)"**.
        *   *Pourquoi ?* K3s et certains clients NFS utilisent des ports aléatoires élevés. Sans ça, le montage échoue.

**Prérequis Client (Serveur K3s) :**
Le paquet `nfs-common` doit être installé sur tous les nœuds du cluster.
*   *Symptôme si absent* : Le pod reste en `ContainerCreating` avec l'erreur `mount failed: exit status 32 ... bad option`.
*   *Solution* : Installé automatiquement via le playbook Ansible (`sre-lab-provisioning`).

**Utilisation :**
Une fois déployé, une `StorageClass` nommée `nfs-client` est créée et définie par défaut.
Tout PVC sans classe spécifique utilisera le NAS.

> **Note sur la StorageClass par défaut** :
> K3s installe par défaut `local-path` (stockage sur le disque du serveur).
> Pour éviter les conflits, nous avons désactivé `local-path` comme classe par défaut afin que `nfs-client` soit le seul choix implicite.
> Commande utilisée : `kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'`

### 2. Monitoring (`monitoring/`)

Stack complète d'observabilité :
*   **Prometheus** : Collecte des métriques.
*   **Grafana** : Visualisation (Dashboards).
*   **Loki** : Agrégation des logs.

### 3. Sécurité (`security/`)

Gestion des secrets et de l'identité :
*   **Vault** : Coffre-fort pour les secrets.
*   **External Secrets** : Synchronise les secrets de Vault vers Kubernetes.

---

## Problèmes Connus & Solutions (Troubleshooting)

### 1. Le problème de "l'œuf et la poule" (CRDs)

**Symptôme :**
Lors du premier déploiement de l'infrastructure, ArgoCD échoue avec une erreur indiquant que la ressource `ClusterSecretStore` (ou `ExternalSecret`) est inconnue.

**Cause :**
L'application `infrastructure` tente de déployer simultanément :
1.  L'application `external-secrets` (qui installe les CRDs).
2.  Le fichier `secrets-config.yaml` (qui utilise ces CRDs).
Kubernetes rejette le fichier de configuration car la définition (CRD) n'est pas encore prête.

**Solution (Procédure de Bootstrap) :**
1.  Commenter le contenu du fichier `infrastructure/security/secrets-config.yaml`.
2.  Pousser sur Git et laisser ArgoCD synchroniser (cela installe les CRDs via `external-secrets`).
3.  Une fois vert, décommenter le fichier et pousser à nouveau.

### 2. ArgoCD Ingress & TLS (Internal Server Error)

**Symptôme :**
L'accès à `https://argocd.local` renvoie une erreur `500 Internal Server Error` via Traefik.

**Cause :**
ArgoCD utilise par défaut un certificat auto-signé pour son interface web. Traefik (le proxy Ingress) refuse de se connecter au backend ArgoCD car il ne fait pas confiance à ce certificat.

**Solution :**
Nous avons configuré ArgoCD pour fonctionner en mode **HTTP (Insecure)** à l'intérieur du cluster, tout en laissant Traefik gérer le HTTPS pour l'extérieur.

1.  **Patch ArgoCD** : Ajouter `--insecure` aux arguments du serveur.
    ```bash
    kubectl patch deployment argocd-server -n argocd --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--insecure"}]'
    ```
2.  **Ingress** : Configurer l'Ingress pour pointer vers le port 80 (HTTP) au lieu de 443.

### 3. Accès DNS Local (`.local`)

**Symptôme :**
Impossible d'accéder à `https://argocd.local` depuis le navigateur Windows, alors que `curl` fonctionne dans WSL.

**Cause :**
WSL2 a sa propre stack réseau. Modifier `/etc/hosts` dans WSL n'affecte pas Windows.

**Solution :**
1.  **WSL/Linux** : Le script `start_lab.sh` met à jour `/etc/hosts` automatiquement.
2.  **Windows** : Il faut ajouter manuellement l'entrée dans `C:\Windows\System32\drivers\etc\hosts` (en tant qu'Admin) :
    ```text
    192.168.1.120 argocd.local grafana.local prometheus.local loki.local
    ```
