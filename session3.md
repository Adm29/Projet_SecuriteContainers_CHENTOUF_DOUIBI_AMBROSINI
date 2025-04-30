# Compte rendu - SESSION 3 : Kubernetes

## Partie 1 : - Déploiement d'un cluster avec Kind

1. **Installation des outils nécessaires**
```bash
sudo apt install docker.io
go install sigs.k8s.io/kind@latest
sudo mv ~/go/bin/kind /usr/local/bin/
sudo apt install kubectl
```

2. **Création d’un cluster avec 2 master et 2 workers**

**Fichier `kind-cluster.yaml`** :
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: control-plane
  - role: worker
  - role: worker
```

**Commande :**
```bash
kind create cluster --name kind --config kind-cluster.yaml
```

3. **Vérification de l’état du cluster**
```bash
kubectl get nodes
```

4. **Liste des namespaces**
```bash
kubectl get namespaces
```

5. **Version de Kubernetes déployée**
```bash
kubectl version --short
```

### Explications des commandes :

- sudo usermod -aG docker $USER : ajoute l’utilisateur courant au groupe docker. Cette commande permet de pouvoir lancer des conteneurs Docker sans avoir à utiliser les privilèges administrateur (root).
  
- curl -Lo ./kind https://kind.sigs.k8s.io/dl/VERSION/kind-linux-amd64 puis chmod +x ./kind puis sudo mv ./kind /usr/local/bin/kind : télécharge le binaire Kind pour Linux, le rend exécutable, puis le déplace dans un répertoire système.
  
- sudo snap install kubectl --classic : installe l’outil kubectl via Snap. Kubectl est l’interface en ligne de commande pour piloter Kubernetes.
  
- kind create cluster : crée un cluster Kubernetes local en utilisant Kind. Cette commande démarre un ou plusieurs conteneurs Docker qui joueront le rôle de nœuds du cluster (par défaut, un seul nœud control-plane). En quelques secondes, on obtient un cluster nommé kind prêt à l’emploi.
  
---

## Partie 2 – RBAC

### 1. Création d’un namespace
```bash
kubectl create ns test-rbac
```

### 2. Déploiement d’un Pod dans ce namespace

**Fichier `mon-pod.yaml` :**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: test-rbac
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f mon-pod.yaml
```

### 3. Affichage des logs du pod
```bash
kubectl logs nginx -n test-rbac
```

### 4. Création d’un rôle RBAC

**Fichier `role-pod-reader.yaml` :**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test-rbac
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

```bash
kubectl apply -f role-pod-reader.yaml
```

### 5. Afficher le rôle créé
```bash
kubectl get role pod-reader -n test-rbac -o yaml
```

### 6. Création du RoleBinding

**Fichier `rolebinding-pod-reader.yaml` :**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: test-rbac
subjects:
- kind: User
  name: titi
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rolebinding-pod-reader.yaml
```

### 7. Création de l'utilisateur fictif `titi`
```bash
docker cp kind-control-plane:/etc/kubernetes/pki/ca.crt .
docker cp kind-control-plane:/etc/kubernetes/pki/ca.key .

openssl genrsa -out titi.key 2048
openssl req -new -key titi.key -out titi.csr -subj "/CN=titi"
openssl x509 -req -in titi.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out titi.crt -days 365
```

### 8. Ajouter l’utilisateur dans le contexte kubeconfig
```bash
kubectl config set-credentials titi \\
  --client-certificate=titi.crt \\
  --client-key=titi.key

kubectl config set-context titi-context \\
  --cluster=kind-kind \\
  --namespace=test-rbac \\
  --user=titi

kubectl config use-context titi-context
```

### Test des permissions :

- **Lister les pods :** OK
- **Créer un pod :** Interdit – message `Forbidden`
- **Retour admin :**
```bash
kubectl config use-context kind-kind
```

---

## Partie 3 – Scan de sécurité avec Kube-Bench

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job.batch/kube-bench -n kube-system
```

### Résumé :
- Vérifie la conformité CIS.
- Signale les mauvaises configurations (root, audit, ports...).

---

## Partie 4 – Détection avec Falco

### Installation Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Déploiement Falco
```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
kubectl create ns falco
helm -n falco install falco falcosecurity/falco --set falcosidekick.enabled=true --set falcosidekick.webui.enabled=true
```

### Interface Falco
```bash
kubectl get pods -n falco
kubectl port-forward svc/falco-falcosidekick-ui 2802:2802 -n falco
# http://127.0.0.1:2802
```

### Génération d’activités

**Fichier `mon-pod.yml` :**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: front
  labels:
    app: front
spec:
  containers:
  - name: front
    image: alpine
    command: ["/bin/sh", "-c", "sleep 1d"]
```

```bash
kubectl apply -f mon-pod.yml
kubectl exec -it front -- sh
```

**Actions dans le pod :**
```bash
apk add curl
curl -k http://10.96.0.1:80
```

### Résultats :
Nous retrouvons une alerte générée par Falco (Warning ou Critical).
- Règle : accès non autorisé à l’API Kubernetes.

---
