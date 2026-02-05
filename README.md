# Kubernetes

> 📌 NOTE : Pour l'installation du cluster, veuillez suivre la documentation dédiée :  
> 👉 [Install_Cluster.md](Install_Cluster.md)

Description
-----------
Ce dépôt contient des fichiers et scripts liés à la configuration et à la gestion d'un cluster Kubernetes. Le point d'entrée principal pour l'installation est le fichier `Install_Cluster.md` (lien ci‑dessus).

Prérequis
---------
- Git
- Un environnement (machine physique ou VM) compatible avec Kubernetes
- kubectl (version compatible avec la version de Kubernetes choisie)
- (Optionnel) outils d'automatisation : kubeadm, kind, minikube, kops, terraform, etc.

Usage rapide
-----------
1. Ouvrez le fichier d'installation : [Install_Cluster.md](Install_Cluster.md)  
2. Suivez les étapes d'installation et de configuration décrites dans ce fichier.  
3. Après installation, utilisez `kubectl` pour vérifier l'état du cluster :
   - kubectl get nodes
   - kubectl get pods --all-namespaces
