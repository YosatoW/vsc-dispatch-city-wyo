# Dispatch City – Block 03 und 04

Dieses Repository dokumentiert den Aufbau von **Dispatch City ab Block 03**. Der aktuelle Projektstand umfasst:

- Block 03: Deployments, Services und ConfigMaps
- Block 04: Ingress und externe Zugriffe

## Releases

- `v1.0.0`: Block 03 – Foundation mit Deployments, Services und ConfigMaps
- `v1.0.4`: Block 04 – Ingress, Traefik, zwei Dashboard-Replikas und Self-Healing

# Schnellstart

Dieser Abschnitt enthält nur die Schritte, die nötig sind, um den aktuellen Stand von Block 04 zu starten. Die ausführliche Beschreibung der umgesetzten Arbeiten folgt weiter unten.

## Erstinstallation auf einem neuen Rechner

### 1. Voraussetzungen

Installiert sein müssen:

- Docker Desktop
- Git
- k3d
- kubectl
- PowerShell unter Windows oder Bash, WSL beziehungsweise Git Bash
- Browser

Versionen prüfen:

```powershell
docker version
k3d version
kubectl version --client
git --version
```

### 2. Repository klonen

```powershell
git clone https://github.com/YosatoW/vsc-dispatch-city.git
cd vsc-dispatch-city
```

### 3. Kubernetes-Cluster erstellen

Docker Desktop muss vollständig gestartet sein.

Prüfen, ob der Cluster bereits vorhanden ist:

```powershell
k3d cluster list
```

Falls `teko-k8s` noch nicht existiert:

```powershell
k3d cluster create teko-k8s --agents 2 --wait
```

Kontext setzen und Nodes prüfen:

```powershell
kubectl config use-context k3d-teko-k8s
kubectl get nodes -o wide
```

Erwartet werden ein Server-Node und zwei Agent-Nodes im Status `Ready`.

### 4. Images bauen

```powershell
docker build -t food-delivery-control-api:local --build-arg SERVICE=control-api -f build/go-service.Dockerfile .
docker build -t food-delivery-dashboard:local ./apps/dashboard
```

### 5. Images in den k3d-Cluster importieren

```powershell
k3d image import -c teko-k8s food-delivery-control-api:local food-delivery-dashboard:local
```

### 6. Block 04 deployen

```powershell
kubectl apply -k ./deploy/overlays/block-04-ingress
kubectl -n food-delivery rollout status deployment/control-api --timeout=180s
kubectl -n food-delivery rollout status deployment/dashboard --timeout=180s
```

Status prüfen:

```powershell
kubectl -n food-delivery get deployment,pods,service,ingress
```

Erwarteter Zustand:

```text
control-api   1/1
dashboard     2/2
Ingress       traefik
```

### 7. Anwendung öffnen

In einem separaten Terminal:

```powershell
kubectl -n kube-system rollout status deployment/traefik --timeout=180s
kubectl -n kube-system port-forward service/traefik 8080:80
```

Das Terminal muss geöffnet bleiben.

Browser:

```text
http://127.0.0.1:8080/
```

Falls Port `8080` belegt ist:

```powershell
kubectl -n kube-system port-forward service/traefik 8081:80
```

Dann lautet die Adresse:

```text
http://127.0.0.1:8081/
```

### 8. Funktion prüfen

Bei Port `8080`:

```powershell
(Invoke-WebRequest -UseBasicParsing -Uri "http://127.0.0.1:8080/").StatusCode
(Invoke-WebRequest -UseBasicParsing -Uri "http://127.0.0.1:8080/api/v1/snapshot").StatusCode
(Invoke-WebRequest -UseBasicParsing -Uri "http://127.0.0.1:8080/health/ready").StatusCode
```

Erwartet:

```text
200
200
200
```

## Neustart bei bereits eingerichteter Umgebung

Wenn Cluster, Images und Kubernetes-Ressourcen bereits vorhanden sind, müssen die Images nicht neu gebaut und die Manifeste nicht erneut installiert werden.

Docker Desktop starten und danach ausführen:

```powershell
cd vsc-dispatch-city
k3d cluster start teko-k8s
kubectl config use-context k3d-teko-k8s
kubectl -n food-delivery get pods
```

Falls der Cluster bereits läuft, kann `k3d cluster start teko-k8s` übersprungen werden.

Anwendung in einem separaten Terminal öffnen:

```powershell
kubectl -n kube-system port-forward service/traefik 8080:80
```

Browser:

```text
http://127.0.0.1:8080/
```

## Umgebung beenden

Port-Forward im entsprechenden Terminal mit `Ctrl + C` beenden.

Cluster stoppen, ohne ihn zu löschen:

```powershell
k3d cluster stop teko-k8s
```

Später kann der Cluster mit `k3d cluster start teko-k8s` wieder gestartet werden.

# Was wurde umgesetzt?

## Block 03 – Deployments, Services und ConfigMaps

- Dispatch-City-Foundation `v1.0.0` übernommen
- Images für Control API und Dashboard gebaut
- Images in den k3d-Cluster importiert
- Kubernetes-Manifeste mit Kustomize gerendert und angewendet
- Namespace `food-delivery` verwendet
- Control API und Dashboard als Deployments betrieben
- Services, EndpointSlices und Kubernetes-DNS geprüft
- ConfigMap `simulation-config` verwendet
- Readiness- und Liveness-Probes geprüft
- Änderungen an `TICK_MS` durch einen Rollout übernommen

## Block 04 – Ingress und externe Zugriffe

- Block-4-Erweiterung aus Release `v1.1.1` integriert
- Projektstand als Release `v1.0.4` markiert
- Ingress `food-delivery` mit Ingress-Klasse `traefik` erstellt
- Dashboard auf zwei Replikas skaliert
- gemeinsames Path Routing eingerichtet
- Dashboard, API und Health-Endpunkt über denselben Einstiegspunkt getestet
- Load Balancing zwischen zwei Dashboard-Pods sichtbar gemacht
- Self-Healing durch Löschen eines Dashboard-Pods geprüft

## Architektur

```text
Browser / Client
       |
       | HTTP, zum Beispiel localhost:8080
       v
Traefik Ingress Controller
       |
       v
Ingress food-delivery
       |
       +-- /         -> dashboard:3000 -> Dashboard-Pod A oder B
       +-- /api      -> control-api:8080
       +-- /health   -> control-api:8080
       +-- /metrics  -> control-api:8080
```

# Technische Prüfungen

## Block 03 rendern und deployen

```powershell
kubectl kustomize ./deploy/overlays/block-03-standalone
kubectl apply -k ./deploy/overlays/block-03-standalone
kubectl -n food-delivery rollout status deployment/control-api
kubectl -n food-delivery rollout status deployment/dashboard
kubectl -n food-delivery get deploy,pods,svc,cm -o wide
```

## Service, EndpointSlice und DNS prüfen

```powershell
kubectl -n food-delivery get svc,endpointslice
kubectl -n food-delivery get pods --show-labels
kubectl -n food-delivery get endpointslice -l kubernetes.io/service-name=control-api
```

Health-Endpunkt aus einem temporären Pod prüfen:

```powershell
kubectl run dns-test --image=curlimages/curl:8.8.0 -n food-delivery --restart=Never --rm -i -- curl -fsS http://control-api:8080/health/ready
```

Vollständiger interner DNS-Name:

```text
http://control-api.food-delivery.svc.cluster.local:8080/health/ready
```

## ConfigMap und Rollout prüfen

```powershell
kubectl diff -k ./deploy/overlays/block-03-standalone
kubectl apply -k ./deploy/overlays/block-03-standalone
kubectl -n food-delivery rollout restart deployment/control-api
kubectl -n food-delivery rollout status deployment/control-api
kubectl -n food-delivery logs deployment/control-api
```

## Block 04 rendern

```powershell
kubectl kustomize ./deploy/overlays/block-04-ingress
```

Erwartete Kerneinstellungen:

- Ingress: `food-delivery`
- Ingress-Klasse: `traefik`
- Dashboard-Replikas: `2`
- Control-API-Replikas: `1`

## Load Balancing prüfen

```powershell
1..20 | ForEach-Object {
    (Invoke-RestMethod -Uri "http://127.0.0.1:8080/ui-instance" -DisableKeepAlive).instance
} | Sort-Object -Unique
```

EndpointSlice prüfen:

```powershell
kubectl -n food-delivery get endpointslice -l kubernetes.io/service-name=dashboard -o wide
```

## Self-Healing prüfen

Pods beobachten:

```powershell
kubectl -n food-delivery get pods -l app.kubernetes.io/name=dashboard -w
```

In einem zweiten Terminal einen Pod löschen:

```powershell
$pod = kubectl -n food-delivery get pod -l app.kubernetes.io/name=dashboard -o jsonpath='{.items[0].metadata.name}'
kubectl -n food-delivery delete pod $pod
kubectl -n food-delivery rollout status deployment/dashboard --timeout=180s
```

Kubernetes erstellt automatisch einen Ersatz-Pod und stellt wieder zwei Dashboard-Replikas bereit.

# Diagnose

```powershell
k3d cluster list
kubectl config current-context
kubectl get nodes -o wide
kubectl -n food-delivery get deploy,pods,svc,endpointslice,ingress,cm -o wide
kubectl -n food-delivery get events --sort-by='.lastTimestamp'
kubectl -n food-delivery describe ingress food-delivery
kubectl -n food-delivery logs deployment/control-api
```

## Häufige Fehler

### Port bereits belegt

Einen anderen lokalen Port verwenden, zum Beispiel `8081:80`.

### ImagePullBackOff

Images erneut bauen und in `teko-k8s` importieren.

### Webseite nicht erreichbar

- Docker Desktop prüfen
- Cluster `teko-k8s` prüfen
- Kontext `k3d-teko-k8s` setzen
- Pods und Rollouts prüfen
- Traefik-Port-Forward neu starten

### 404 über Traefik

```powershell
kubectl -n food-delivery describe ingress food-delivery
```

# Git-Workflow

Vor Arbeitsbeginn:

```powershell
git pull
```

Änderungen hochladen:

```powershell
git status
git add .
git commit -m "Describe change"
git push
```

## Release-Tags

```text
v1.0.0  Block 03
v1.0.4  Block 04
```

# Aufräumen

Nur die Dispatch-City-Ressourcen entfernen:

```powershell
kubectl delete namespace food-delivery
```

Vollständiger Reset inklusive Cluster:

```powershell
k3d cluster delete teko-k8s
```
