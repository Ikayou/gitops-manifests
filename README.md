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

## Nützliche Befehle zur Verwaltung
```bash
# Externe IP-Adresse des Load Balancers abrufen
kubectl get svc backend-service

# Status der Pods prüfen
kubectl get pods -n default