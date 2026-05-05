# GitOps Manifests (CD)

Dieses Repository verwaltet die Ressourcen-Definitionen (Kubernetes-Manifeste) für den AKS-Cluster. Argo CD überwacht dieses Repository und hält den Zustand des Clusters synchron mit dem Code.

## Externer Zugriff (Load Balancer)
Diese Anwendung ist so konfiguriert, dass sie über einen **Azure Load Balancer** direkt aus dem Internet erreichbar ist. 
- In der `service.yaml` ist der Typ `LoadBalancer` definiert.
- Azure weist automatisch eine öffentliche IP-Adresse zu, die den Datenverkehr an die Flask-App (Port 8000) weiterleitet.

## Verzeichnisstruktur
- **apps/backend/**: Definitionen für die Backend-App
  - `deployment.yaml`: Anzahl der Pods, verwendetes Image und Container-Einstellungen
  - `service.yaml`: **Expositions-Konfiguration via Load Balancer (Port 80)**
- **argocd/**: Konfiguration für Argo CD
  - `backend-app.yaml`: Definition der Argo CD Application Ressource

## GitOps Workflow
1. Die Datei `apps/backend/deployment.yaml` wird aktualisiert (manuell oder durch CI).
2. **Argo CD** erkennt die Differenz und führt einen Sync durch.
3. Die Anwendung wird im Cluster aktualisiert und bleibt über die **externe IP-Adresse** des Load Balancers erreichbar.

## Nützliche Befehle zur Verwaltung
```bash
# Externe IP-Adresse des Load Balancers abrufen
kubectl get svc python-backend-service

# Status der Pods prüfen
kubectl get pods -n default