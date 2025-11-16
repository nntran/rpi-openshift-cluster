# Déploiement MicroShift ARM + GitOps complet

Ce guide décrit comment déployer un cluster MicroShift sur RPI5 (3 masters, 3 workers), avec LB HAProxy sur RPI4, Traefik Ingress, cert-manager avec Let's Encrypt DNS-01 (Cloudflare), Tekton, Gitea, Nexus, Harbor et Vault via ArgoCD.

## Prérequis
- RPI5 × 6 (3 masters, 3 workers), RPI4 × 1 (LB)
- OS ARM64 (Ubuntu 22.04 recommandé)
- Ansible 2.15+ sur une machine de contrôle
- Accès SSH key-based vers tous les nodes
- Token Cloudflare pour DNS-01 (Zone:DNS:Edit)
- Docker ou Podman sur la machine Ansible (pour tests multi-arch builds)

## Structure du repo Ansible

```
rpi-openshift-cluster/
├── playbooks/site.yml
├── inventory/hosts.ini
├── group_vars/all.yml
├── roles/
│ ├── common
│ ├── kernel_tune
│ ├── prereqs
│ ├── microshift
│ ├── lb_haproxy
│ ├── argocd
│ ├── gitops-bootstrap
│ └── secrets-bootstrap
```

## Étapes de déploiement

### 1. Préparer le fichier `group_vars/all/vault.yml`

- Ajouter toutes les valeurs sensibles (Cloudflare token, Gitea admin, Docker registry, email ACME)
- Chiffrer avec `ansible-vault encrypt group_vars/all/vault.yml`

### 2. Bootstrap nodes

```bash
ansible-playbook playbooks/site.yml -l all --tags bootstrap
```

### 3. Installer MicroShift sur masters & workers

```sh
ansible-playbook playbooks/site.yml -l masters,workers --tags microshift

```

### 4. Configurer LoadBalancer HAProxy sur RPI4

```sh
ansible-playbook playbooks/site.yml -l lb --tags lb_haproxy

```

### 5. Installer ArgoCD

```sh
ansible-playbook playbooks/site.yml -l masters[0] --tags argocd

```

### 6. Créer les secrets k8s via Ansible

```sh
ansible-playbook playbooks/site.yml -l masters[0] --tags secrets-bootstrap

```

### 7. Déployer les applications via ArgoCD

- Pousser infra-apps.git sur le repo
- Synchroniser ArgoCD Applications :
  - Traefik
  - cert-manager
  - ClusterIssuer (staging → test)
  - Tekton
  - Gitea
  - Nexus
  - Harbor
  - Vault

### 8. Vérification des certificats TLS

```sh
kubectl get certificate -A
kubectl describe certificat <nom>
```
Tester d’abord en staging, puis passer à production en modifiant ClusterIssuer.

### 9. Déployer les images multi-arch si nécessaire

- Utiliser le script tools/build_multiarch.sh ou GitHub Actions ci-dessous
- Mettre à jour Helm chart values avec la nouvelle image.

## Notes & recommandations

- Utiliser kubectl get pods -n <namespace> pour vérifier la santé.
- Toujours tester staging Let’s Encrypt avant prod pour éviter les rate limits.
- Vérifier compatibilité ARM64 pour Nexus et autres images.
