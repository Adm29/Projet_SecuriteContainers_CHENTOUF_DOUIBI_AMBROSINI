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

- **Lister les pods :**
  
Le retour affiche correctement la liste des pods présents dans le namespace test-rbac. Cela est normal car l’utilisateur titi possède un Role qui l’autorise à utiliser les verbes get et list sur la ressource pods. Il a donc les droits de lecture dans ce namespace.

- **Créer un pod :**
Non. Si l’on tente de créer un nouveau pod on a un message d'erreur :

```bash
Error from server (Forbidden): error when creating "mon-pod.yaml": pods is forbidden: User "titi" cannot create resource "pods" in API group "" in the namespace "test-rbac"
```
Cette erreur indique clairement que l’utilisateur titi ne possède pas les droits nécessaires pour créer un pod.

- **Retour admin :**
Pour retrouver tous les privilèges administrateur, il faut repasser dans le contexte d’origine à l’aide de la commande suivante :

```bash
kubectl config use-context kind-kind
```
---

## Partie 3 – Scan de sécurité avec Kube-Bench

Nous avons utilisé Kube-Bench, un outil open source développé par Aqua Security, pour analyser la configuration de sécurité de notre cluster Kubernetes. Cet outil vérifie automatiquement si les composants du cluster respectent les bonnes pratiques de sécurité.

### Etape 1 : Création du Job Kube-Bench

Pour lancer le scan, nous avons créé un Job Kubernetes à partir d’un manifeste YAML fourni par la documentation officielle :

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```
### Etape 2 : Lecture des résultats du Scan

Une fois le job terminé, nous avons affiché les logs pour consulter le rapport de sécurité :
```bash
kubectl logs job.batch/kube-bench -n kube-system
```
Le rapport fournit un ensemble de tests classés par composant :
- API Server
- Controller Manager
- Scheduler
- etcd
- RBAC et Authentication

Chaque test est marqué par un statut :
- PASS : la configuration est conforme aux bonnes pratiques,
- FAIL : une faiblesse de configuration est détectée,
- WARN : attention, configuration potentiellement risquée,
- INFO : informations supplémentaires sans impact direct.

**Ce type d’analyse est essentiel pour garantir un niveau de sécurité élevé dans un environnement Kubernetes.**

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

Une fois Falco installé et déployé, nous avons cherché à déclencher des alertes de sécurité pour vérifier que l’outil fonctionne correctement.

Nous avons commencé par exécuter un shell interactif dans le pod nommé front à l’aide de la commande suivante :

```bash
kubectl apply -f mon-pod.yml
kubectl exec -it front -- sh
```

Après avoir ouvert ce shell, nous avons pu observer qu’une alerte de sécurité avait bien été générée. Cette alerte signalait une “shell interactive démarrée dans un conteneur” : 

"Shell was spawned in a container"

Toujours dans le shell du pod, nous avons ensuite simulé un comportement encore plus suspect : un appel manuel vers l’API Kubernetes. 

Pour cela, nous avons d’abord installé curl puis nous avons exécuté une requête HTTP vers l’adresse de service de l’API :

```bash
apk add curl
curl -k http://10.96.0.1:80
```

Une fois cette commande exécutée, une deuxième alerte Falco a été déclenchée. Cette alerte indique qu’un processus à l’intérieur d’un conteneur a tenté d’accéder à l’API Kubernetes, ce qui est considéré comme une activité suspecte. Une telle action peut suggérer une tentative d’exfiltration de données ou une exploration non autorisée du cluster. Falco détecte ce comportement en surveillant les connexions réseau établies depuis les conteneurs vers l’adresse de l’API Kubernetes.

**La priorité de cette alerte est  “Critical. 
La règle déclenchée est :

```bash
Contact K8s API Server From Containers
```
**Le message complet :**

```bash
"output": "Contact K8s API Server From Container (user=root user_loginuid=0 command=curl -k http://10.96.0.1:80 container_id=... image=alpine)",
  "priority": "Critical",
  "rule": "Contact K8s API Server From Container",
  "time": "2025-04-30T14:55:00.000Z",
  "output_fields": {
    "user.name": "root",
    "proc.cmdline": "curl -k http://10.96.0.1:80",
    "container.id": "abc123def456",
    "container.image.repository": "alpine"
```
