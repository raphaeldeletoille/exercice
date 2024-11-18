# Exercice: Déploiement d'une application sur Azure Kubernetes Service (AKS)

## Prérequis
- Git
- Docker Desktop
- Kubectl 1.30
- Azure Subscription

## Étapes

**Déployer les prérequis sur Azure** (en Terraform)

Créer un Azure Container Registry (ACR). Donnez à votre compte utilisateur les droits ACR Push

Créer un cluster Kubernetes (Standard_D2_v2) en sku_tier "Standard". L'identité pour votre Cluster doit être "acridentity" (déjà créée sur Azure). Cette identité doit être utilisée dans le bloc identity (User Assigned) et kubelet identity.

**Cloner le dépôt Git**
   ```bash
   git clone https://github.com/raphaeldeletoille/exercice
   cd exercice
  ```
---
**Déployer votre application localement**

Lancer les conteneurs avec Docker Compose

  ```
docker compose -f docker-compose-quickstart.yml up -d
  ```

Afficher les images Docker créées
  ```
docker images
  ```

Afficher les conteneurs en cours d'exécution
  ```
  docker ps
  ```

Accéder à l'application localement 
  ```
  http://localhost:8080
  ```

Arrêter et supprimer les conteneurs
  ```
  docker compose down
  ```
---
**Déployer votre application sur Azure**

Propagez les permissions distribués par terraform.

Vous pouvez trouver l'acr-resource-id dans JsonView depuis l'interface graphique de votre container registry (Azure Portail) 
 ```
az aks update -n <myAKSCluster> -g <myResourceGroup> --attach-acr <acr-resource-id>
 ```

Construire et pousser les images Docker sur ACR
  ```
az acr build --registry $ACRNAME --image exercice/product-service:latest ./src/product-service/
az acr build --registry $ACRNAME --image exercice/order-service:latest ./src/order-service/
az acr build --registry $ACRNAME --image exercice/store-front:latest ./src/store-front/
  ```

Lister les dépôts dans ACR
az acr repository list --name $ACRNAME --output table

Connecter son terminal au cluster Kubernetes (Vous trouverez comment faire depuis l'interface graphique Azure Portail)

Tester la connection en affichant les serveurs utilisés par Kubernetes
  ```
  kubectl get nodes
  ```

Lister l'url de connection à l'ACR 
  ```
  az acr list --resource-group myResourceGroup --query "[].{acrLoginServer:loginServer}" --output table
  ```

Modifier aks-store-quickstart.yaml avec les images ACR
  ```
containers:
...
- name: order-service
  image: <acrName>.azurecr.io/exercice/order-service:latest
...
- name: product-service
  image: <acrName>.azurecr.io/exercice/product-service:latest
...
- name: store-front
  image: <acrName>.azurecr.io/exercice/store-front:latest

  ```

Appliquer la configuration Kubernetes
  ```
  kubectl apply -f aks-store-quickstart.yaml
  ```

Vérifier les pods déployés
  ```
  kubectl get pods
  ```

Vérifier le service store-front et obtenir l'IP externe
  ```
kubectl get service store-front --watch
  ```

Accéder à votre service via l'IP externe

Scalez votre nombre de Pods pour augmenter la capacité de votre application
  ```
  kubectl scale --replicas=5 deployment.apps/store-front
  ```

Scalez la capacité de Kubernetes est aussi possible 
  ```
az aks scale --resource-group myResourceGroup --name myAKSCluster --node-count 3
  ```
