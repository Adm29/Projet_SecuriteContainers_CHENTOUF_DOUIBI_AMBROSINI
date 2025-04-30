# Compte rendu – Session 2 : Bonnes pratiques de sécurité des conteneurs (Windows 11 powershell + Docker Desktop)

**Date :** 9 avril 2025   

---

## 1. Objectifs

- Mettre en place les meilleures pratiques de sécurisation des conteneurs sous Windows 11 (Docker Desktop + WSL 2).
- Comprendre la gestion des droits, du réseau et des secrets.


---

## 2. Environnement de test

| Élément                    | Version / build          | Commande PowerShell                             |                    
| -------------------------- | ------------------------ | ----------------------------------------------- |
| **Windows**                | 11 Pro 23H2 (22621.4391) | \`systeminfo                                    | 
| **Docker Desktop**         | 28.0.1                   | `docker version --format '{{.Server.Version}}'` |                    
| **WSL 2 (docker-desktop)** | WSL 2                    | `wsl --status`                                  |                    
| **Trivy**                  | 0.61.1                   | `trivy --version`                               |                    
| **Docker Bench Security**  | 48e5c51 (2025‑04)        | `git -C docker-bench-security log -1 --oneline` |                    
| **PowerShell**             | 5.1.22621.4391           | `$PSVersionTable.PSVersion`                     |                    

---

## 3. Gestion réseau et exposition de ports

### 3.1 Réseaux Docker isolés

```powershell
# Créer un réseau bridge isolé
docker network create --driver bridge secure-net
```

Résultat – Figure 1: ![Capture 1](https://github.com/user-attachments/assets/f255c67d-8abd-4e76-a26e-1bdde908e4a5)


```powershell
# Lancer Nginx dans secure-net
docker run -d --name web --network secure-net nginx
```

Résultat – Figure 2 :![Capture 2](https://github.com/user-attachments/assets/2fa93a73-1c70-49c6-8a33-7157027652a8)


```powershell
# Tester l’accès interne
docker run --rm --network secure-net curlimages/curl http://web
```

Résultat – Figure 3 :![Capture 3](https://github.com/user-attachments/assets/e027d97b-32e9-4ccb-8298-4ca6b4375fca)


**Analyse 3.1 :** accès HTTP 200 uniquement depuis le réseau isolé.

### 3.2 Limitation de l’exposition des ports

```powershell
# Exemple de mauvaise pratique
docker run -d -p 8080:80 --name nginx_public nginx
netstat -ano | findstr :8080
```

Résultat – Figure 4 :![Capture 4](https://github.com/user-attachments/assets/10f576a4-f942-41bc-b480-1ac43ed1ef4b)


**Analyse 3.2 :** port 8080 ouvert sur toutes interfaces → risque élevé, préférer reverse-proxy ou réseau interne.

---

## 4. Volumes en lecture seule et audits

### 4.1 Volume read-only

```powershell
# Monter hosts en lecture seule
docker run -it --rm -v C:\Windows\System32\drivers\etc\hosts:/mnt/hosts:ro alpine sh
```

**Dans le conteneur :**

```sh
cat /mnt/hosts        # lecture OK
echo "test" >> /mnt/hosts  # Read-only file system
```

Résultat – Figure 5 :![capture 5](https://github.com/user-attachments/assets/44bee767-ad63-4c64-8e9a-86fcfe3aa23e)


**Analyse 4.1 :** protection efficace : aucun écrit possible dans le fichier hôte.

### 4.2 Audit Docker Bench – Hôte

```powershell
# Audit complet sur l’hôte
docker run --rm --net host --pid host --cap-add audit_control \
  -v //var/run/docker.sock:/var/run/docker.sock \
  docker/docker-bench-security:latest
```

Résultat – Figure 6 :![capture 6](https://github.com/user-attachments/assets/a733d3d1-1fc2-48ec-91aa-5fce2a72ca06) 

**Analyse 4.2 :** 105 checks exécutés, Score 10/280 (nombreux tests SKIPPED sur Docker Desktop).

### 4.3 Audit Docker Bench – DVWA

```powershell
# Lancer conteneur vulnérable
docker run -d --name dvwa vulnerables/web-dvwa
# Audit ciblé sur DVWA
docker run --rm --net host --pid host --cap-add audit_control \
  -v //var/run/docker.sock:/var/run/docker.sock \
  docker/docker-bench-security:latest -i dvwa
```

Résultat – Figure 7 :![Capture 7](https://github.com/user-attachments/assets/c44fd501-57cb-489b-af22-45aaf6a4a03f)

**Analyse 4.3 :** DVWA tourne en root sans Seccomp/AppArmor, ressources illimitées, health‑check absent.

---

## 5. Gestion des secrets avec Vault

### 5.1 Lancement de Vault en mode Dev

```powershell
# Lancer Vault avec token root
docker run --cap-add IPC_LOCK -e VAULT_DEV_ROOT_TOKEN_ID=root \
  -p 8200:8200 --name vault hashicorp/vault:1.14
```

**(Optionnel : warning "failed to lock memory" sans impact en mode dev.)**

### 5.2 Création du secret via l’UI

1. Accéder à [http://localhost:8200](http://localhost:8200)
2. Se connecter avec token `root`
3. **Secrets Engines ➔** **Enable new** : choisir **KV v2**, Path `containers`
4. **containers » Create secret** : Path `mon-secret`, clé `apiKey`, valeur `sk_live_DEMO123`

### 5.3 Lecture du secret depuis un conteneur Alpine

```powershell
# Lance un conteneur Alpine interactif
docker run -it --rm --name alpine-vault curlimages/curl sh
```

Dans le conteneur :

```sh
export VAULT_ADDR=http://host.docker.internal:8200
curl -s -H "X-Vault-Token: root" $VAULT_ADDR/v1/containers/mon-secret
```

Résultat – Figure 8 :![Capture 8 Bis](https://github.com/user-attachments/assets/934858f5-f54d-4c7a-983a-dda536e2a940)


**Analyse 5 :** le secret est stocké et récupéré dynamiquement, aucune fuite dans l’image.

---

## 6. Inspection de l’image *ety92/demo**:v1*

```powershell
# Recherche de clés/API codées en dur
docker pull ety92/demo:v1
$id = docker images --filter reference=ety92/demo:v1 --format '{{.ID}}'
docker run --rm $id sh -c "env | grep -Ei 'key|token|secret'"
```

**Résultat :** Aucune clé détectée

**Analyse 6 :** image propre, bonnes pratiques suivies.

---

## 7. Conclusions et recommandations

1. Intégrer Trivy + Gitleaks (scans secrets) dans le pipeline CI/CD.
2. Interdire l’exposition de ports directs ; utiliser des reverse-proxies et réseaux internes.
3. Centraliser la gestion des secrets via Vault ou Docker Secrets.
4. Exécuter Docker Bench nightly sur runners Linux (score visé ≥ 260/280).
5. Appliquer par défaut des profils Seccomp et AppArmor/LCOW.

---

###

