# GitOps Manifests (CD)

Dieses Repository verwaltet die Ressourcen-Definitionen (Kubernetes-Manifeste) für den AKS-Cluster. Argo CD überwacht dieses Repository und hält den Zustand des Clusters synchron mit dem Code.

## Externer Zugriff (Load Balancer)
Diese Anwendung ist so konfiguriert, dass sie über einen **Azure Load Balancer** direkt aus dem Internet erreichbar ist. 
- In der `service.yaml` ist der Typ `LoadBalancer` definiert.
- Azure weist automatisch eine öffentliche IP-Adresse zu, die den Datenverkehr an die Flask-App (Port 8000) weiterleitet.

## Verzeichnisstruktur
- **apps/backend/**: Definitionen für die Backend-App
  - **base/**: Gemeinsame Basis-Manifeste (Deployment, Service)
  - **overlays/**: Umgebungsspezifische Anpassungen via **Kustomize**
    - **staging/**: Konfiguration für die Testumgebung (1 Replica)
    - **production/**: Konfiguration für die Produktion (**3 Replicas**, `prd-` Prefix via Patches)

## CI/CD Automatisierung
Dieses Projekt nutzt eine vollständige CI/CD-Pipeline:
1. **GitHub Actions**: Baut das Docker-Image, pusht es in die **Azure Container Registry (ACR)** und aktualisiert automatisch den Image-Tag in den Kustomize-Overlays (`staging` & `production`) mittels `kustomize edit`.
2. **Argo CD**: Überwacht die Overlays und synchronisiert die Änderungen mit dem AKS-Cluster.

## GitOps Workflow
1. Die Datei `apps/backend/deployment.yaml` wird aktualisiert (manuell oder durch CI).
2. **Argo CD** erkennt die Differenz und führt einen Sync durch.
3. Die Anwendung wird im Cluster aktualisiert und bleibt über die **externe IP-Adresse** des Load Balancers erreichbar.

## Troubleshooting / Setup Hinweis
Damit der AKS-Cluster Images aus der ACR ziehen kann, muss die Berechtigung explizit gesetzt werden:
```bash
az aks update --resource-group devsecops-rg --name devsecops-aks --attach-acr acrabcreg2026test
```

## Nützliche Befehle zur Verwaltung
```bash
# Externe IP-Adresse des Load Balancers abrufen
kubectl get svc backend-service

# Status der Pods prüfen
kubectl get pods -n default
```

## Ingress-Controller & Traffic-Management
Wir haben den Zugriff von mehreren `LoadBalancer`-IPs auf eine einzige, zentrale IP-Adresse umgestellt.
*   **NGINX Ingress Controller**: Dient als zentrales Gateway für den gesamten Cluster.
*   **Pfad-basiertes Routing**: Durch die Nutzung von Ingress-Ressourcen und Kustomize-Patches werden Anfragen basierend auf dem URL-Pfad verteilt:
    *   `http://<Global-IP>/staging` ➔ **Staging-Umgebung** (Namespace: `staging`)
    *   `http://<Global-IP>/prod` ➔ **Produktions-Umgebung** (Namespace: `production`)
*   **Rewrite-Target**: Die Annotation `nginx.ingress.kubernetes.io/rewrite-target: /` ermöglicht es der Flask-App, Anfragen zu verarbeiten, ohne dass der Pfad-Präfix im Anwendungscode hartcodiert sein muss.

## Aktualisierte Verzeichnisstruktur
```text
apps/backend/
├── base/
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   ├── service.yaml        # Geändert auf ClusterIP (intern)
│   └── ingress.yaml        # Definition des zentralen Gateways
├── overlays/
|    ├── staging/
|    │   └── kustomization.yaml # Patch für /staging Pfad
|    └── production/
|        ├── kustomization.yaml # Patch für /prod Pfad & Replicas
|        └── patch-replicas.yaml
└── argocd/
     └── backend-app.yaml 
```

## 🔍 Überprüfung der Infrastruktur
Um die zentrale IP-Adresse des Gateways zu ermitteln, nutzen wir:
```bash
kubectl get svc -n ingress-basic
```
---

## Cluster-Komponenten (Einmalige Einrichtung)
Diese Komponenten werden direkt im AKS-Cluster installiert:

*   **Argo CD**: Für das GitOps-basierte Deployment.
    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
*   **NGINX Ingress Controller**: Als zentrales Gateway (via Helm).
    ```bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install ingress-nginx ingress-nginx/ingress-nginx --create-namespace --namespace ingress-basic
    ```
---
