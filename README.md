# PayMyBuddy - De la Conteneurisation au Déploiement Kubernetes

## Description

Ce projet modernise le déploiement de l'application **PayMyBuddy**, une solution de gestion de transactions financières entre amis. Initialement monolithique et déployée manuellement, l'application est désormais conteneurisée avec **Docker**, orchestrée avec **Kubernetes** et utilise **Docker Compose** pour la gestion locale des services.

## Prérequis

Avant de commencer, assurez-vous d'avoir installé les outils suivants :

- **Docker**
- **Docker Compose**
- **Kubernetes (K3s recommandé pour un environnement léger)**

## Installation des Outils

### 1. Installer Docker

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

Vérifiez l'installation :

```bash
docker --version
```

### 2. Installer Docker Compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Vérifiez l'installation :

```bash
docker-compose --version
```

### 3. Installer Kubernetes (K3s)

```bash
curl -sfL https://get.k3s.io | sh -
```

Vérifiez l'installation :

```bash
kubectl get nodes
```

---

## Conteneurisation de l'Application

### 1. Dockerfile Backend (Spring Boot)

Fichier `backend/Dockerfile` :

```dockerfile
FROM amazoncorretto:17-alpine
WORKDIR /app
COPY target/paymybuddy.jar .
EXPOSE 8080
CMD ["java", "-jar", "paymybuddy.jar"]
```

### 2. Dockerfile Base de Données MySQL

Fichier `db/Dockerfile` :

```dockerfile
FROM mysql:8
ENV MYSQL_ROOT_PASSWORD=password
ENV MYSQL_DATABASE=db_paymybuddy
ENV MYSQL_USER=paymybuddy
ENV MYSQL_PASSWORD=paymybuddy
COPY initdb /docker-entrypoint-initdb.d/
```

### 3. Construction et Publication des Images Docker

```bash
docker build -t drsbvb/paymybuddy-backend:latest ./backend
docker build -t drsbvb/paymybuddy-db:latest ./database
docker push drsbvb/paymybuddy-backend:latest
docker push drsbvb/paymybuddy-db:latest
```

---

## Déploiement avec Docker Compose

Fichier `docker-compose.yml` :

```yaml
version: "3.8"
services:
  paymybuddy-db:
    build: ./db
    container_name: paymybuddy-db
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: db_paymybuddy
      MYSQL_USER: paymybuddy
      MYSQL_PASSWORD: paymybuddy
  
  paymybuddy-backend:
    build: ./backend
    container_name: paymybuddy-backend
    ports:
      - "8080:8080"
    depends_on:
      - paymybuddy-db
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://paymybuddy-db:3306/db_paymybuddy
      SPRING_DATASOURCE_USERNAME: paymybuddy
      SPRING_DATASOURCE_PASSWORD: paymybuddy

volumes:
  db_data:
```

Démarrer l'application :

```bash
docker-compose up -d
```

Vérifier l'exécution des conteneurs :

```bash
docker ps
```

---

## Déploiement avec Kubernetes

### 1. Déploiement de MySQL

Fichier `mysql-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          envFrom:
            - configMapRef:
                name: mysql-config
            - secretRef:
                name: mysql-secret
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
```

Déploiement :

```bash
kubectl apply -f mysql-deployment.yaml
```

### 2. Déploiement du Backend

Fichier `backend-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: drsbvb/paymybuddy-backend:latest
          envFrom:
            - configMapRef:
                name: backend-config
          env:
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
          ports:
            - containerPort: 8080
```

Déploiement :

```bash
kubectl apply -f backend-deployment.yaml
```

### 3. Déploiement du ConfigMap

Fichier `config.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  DATABASE_HOST: mysql
  DATABASE_USER: root
  SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/paymybuddy
  SPRING_DATASOURCE_USERNAME: root
  SPRING_JPA_HIBERNATE_DDL_AUTO: update
```

Déploiement :

```bash
kubectl apply -f config.yaml
```

### 4. Déploiement du Secret

Fichier `secret.yaml` :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: cGFzc3dvcmQ= # Base64 de "password"
```

Déploiement :

```bash
kubectl apply -f secret.yaml
```

---

## Vérification et Accès

### 1. Vérifier les Pods

```bash
kubectl get pods
```

### 2. Accéder au Backend

```bash
kubectl get svc
curl http://<NODE-IP>:30080/
```



