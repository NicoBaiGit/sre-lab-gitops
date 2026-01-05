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

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```
