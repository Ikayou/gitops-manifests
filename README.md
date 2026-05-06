
## 🛠 Erweiterte GitOps-Konfiguration (Kustomize)
Wir nutzen **Kustomize**, um dieselbe Codebasis für verschiedene Umgebungen zu verwenden, während wir spezifische Anpassungen (z. B. Anzahl der Replicas) vornehmen.

### Verzeichnisstruktur (Overlays)
- **apps/backend/base/**: Gemeinsame Basis-Manifeste (Deployment, Service).
- **apps/backend/overlays/staging/**: Konfiguration für die **Testumgebung** (1 Replica).
- **apps/backend/overlays/production/**: Konfiguration für die **Produktion**.
    - Nutzt `namePrefix: prd-` zur Ressourcentrennung.
    - Nutzt `patch-replicas.yaml`, um die Skalierung auf **3 Replicas** zu erhöhen.

##  CI/CD Automatisierung mit GitHub Actions
Die gesamte Pipeline ist vollständig automatisiert:
1. **CI-Build**: Bei einem Push auf `main` baut GitHub Actions das Docker-Image und pusht es in die **Azure Container Registry (ACR)**.
2. **Manifest Update**: GitHub Actions nutzt `kustomize edit set image`, um den neuen Image-Tag (Commit-SHA) automatisch in die Overlays (`staging` & `production`) zu schreiben.
3. **GitOps Sync**: **Argo CD** erkennt die Änderung im Manifest-Repo und rollt die neue Version ohne manuellen Eingriff auf dem AKS-Cluster aus.

##  Infrastruktur-Anmerkungen (Security & Zugriff)
- **ACR Integration**: Der AKS-Cluster wurde explizit autorisiert, Images aus der ACR zu ziehen:
  ```bash
  az aks update --resource-group devsecops-rg --name devsecops-aks --attach-acr acrabcreg2026test
  ```
- **Namespaces**: Zur sauberen Trennung werden die Anwendungen in den Namespaces `staging` und `production` betrieben.

##  Cleanup
Um Kosten zu sparen, kann die gesamte Infrastruktur wie folgt entfernt werden:
1. **Argo CD Applications löschen**: `kubectl delete -f argocd/`
2. **Terraform Ressourcen zerstören**: `terraform destroy`