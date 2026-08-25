# 1. Dispatch City – Block 03 bis 05

Dieses Repository dokumentiert den Aufbau von **Dispatch City ab Block 03**.

- [1. Dispatch City – Block 03 bis 05](#1-dispatch-city--block-03-bis-05)
  - [1.1. Releases](#11-releases)
- [2. Schnellstart](#2-schnellstart)
  - [2.1. Voraussetzungen](#21-voraussetzungen)
  - [2.2. Erstinstallation auf einem neuen Rechner](#22-erstinstallation-auf-einem-neuen-rechner)
    - [2.2.1. Repository klonen](#221-repository-klonen)
    - [2.2.2. Cluster erstellen](#222-cluster-erstellen)
    - [2.2.3. Block-05-Images bauen](#223-block-05-images-bauen)
    - [2.2.4. Images in k3d importieren](#224-images-in-k3d-importieren)
    - [2.2.5. Block 05 deployen](#225-block-05-deployen)
    - [2.2.6. Anwendung öffnen](#226-anwendung-öffnen)
  - [2.3. Neustart bei bereits eingerichteter Umgebung](#23-neustart-bei-bereits-eingerichteter-umgebung)
  - [2.4. Umgebung beenden](#24-umgebung-beenden)
- [3. Was wurde umgesetzt?](#3-was-wurde-umgesetzt)
  - [3.1. Block 03 – Deployments, Services und ConfigMaps](#31-block-03--deployments-services-und-configmaps)
  - [3.2. Block 04 – Ingress und externe Zugriffe](#32-block-04--ingress-und-externe-zugriffe)
  - [3.3. Block 05 – Messaging mit RabbitMQ](#33-block-05--messaging-mit-rabbitmq)
  - [3.4. Architektur nach Block 05](#34-architektur-nach-block-05)
- [4. Technische Nachweise](#4-technische-nachweise)
  - [4.1. Block 03](#41-block-03)
  - [4.2. Block 04](#42-block-04)
  - [4.3. Block 05](#43-block-05)
    - [4.3.1. Rückstau und Competing Consumers](#431-rückstau-und-competing-consumers)
- [5. Fehlerbehebung](#5-fehlerbehebung)
  - [5.1. RabbitMQ bleibt `0/1` oder startet wiederholt neu](#51-rabbitmq-bleibt-01-oder-startet-wiederholt-neu)
  - [5.2. Port bereits belegt](#52-port-bereits-belegt)
  - [5.3. ImagePullBackOff](#53-imagepullbackoff)
  - [5.4. Diagnose](#54-diagnose)
- [6. Git-Workflow](#6-git-workflow)
- [7. Aufräumen](#7-aufräumen)

## 1.1. Releases

- `v1.0.0`: Block 03 – Deployments, Services und ConfigMaps
- `v1.0.4`: Block 04 – Ingress, Traefik, Load Balancing und Self-Healing
- `v1.0.5`: Block 05 – RabbitMQ, verteilte Worker und Competing Consumers

# 2. Schnellstart

## 2.1. Voraussetzungen

- Docker Desktop
- Git
- k3d
- kubectl
- PowerShell
- Browser

```powershell
docker version
k3d version
kubectl version --client
git --version
```

## 2.2. Erstinstallation auf einem neuen Rechner

### 2.2.1. Repository klonen

```powershell
git clone https://github.com/YosatoW/vsc-dispatch-city-wyo.git
cd vsc-dispatch-city-wyo
```

### 2.2.2. Cluster erstellen

Docker Desktop muss vollständig gestartet sein.

```powershell
k3d cluster list
```

Falls `teko-k8s` noch nicht existiert:

```powershell
k3d cluster create teko-k8s --agents 2 --wait
```

```powershell
kubectl config use-context k3d-teko-k8s
kubectl get nodes -o wide
```

Erwartet werden ein Server-Node und zwei Agent-Nodes im Status `Ready`.

### 2.2.3. Block-05-Images bauen

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\scripts\build-images.ps1
```

Erwartete Images:

```text
food-delivery-control-api:local
food-delivery-customer-simulator:local
food-delivery-courier-simulator:local
food-delivery-order-worker:local
food-delivery-restaurant-worker:local
food-delivery-dashboard:local
```

### 2.2.4. Images in k3d importieren

```powershell
.\scripts\load-images.ps1 -Cluster teko-k8s
```

### 2.2.5. Block 05 deployen

```powershell
kubectl apply -k ./deploy/overlays/block-05-messaging
kubectl -n food-delivery rollout status statefulset/rabbitmq --timeout=300s
kubectl -n food-delivery wait --for=condition=Available deployment --all --timeout=300s
kubectl -n food-delivery wait --for=condition=Ready pod --all --timeout=300s
```

Status prüfen:

```powershell
kubectl -n food-delivery get deploy,statefulset,pods,pvc
kubectl -n food-delivery get configmap simulation-config -o jsonpath='{.data.APP_MODE}'
```

Erwartet werden unter anderem:

```text
RabbitMQ:             1/1 Ready
Dashboard:            2/2 Ready
Order Worker:         2/2 Ready
data-rabbitmq-0:      Bound
APP_MODE:             distributed
```

### 2.2.6. Anwendung öffnen

Terminal 1, Dispatch City über Traefik:

```powershell
kubectl -n kube-system port-forward service/traefik 8081:80
```

Terminal 2, RabbitMQ Management:

```powershell
kubectl -n food-delivery port-forward service/rabbitmq 15672:15672
```

Browser:

```text
Dispatch City:        http://127.0.0.1:8081/
RabbitMQ Management:  http://127.0.0.1:15672/
```

Falls Port `8081` belegt ist, kann ein anderer freier lokaler Port verwendet werden, zum Beispiel `8080:80`.

## 2.3. Neustart bei bereits eingerichteter Umgebung

```powershell
k3d cluster start teko-k8s
kubectl config use-context k3d-teko-k8s
kubectl -n food-delivery get pods
```

Falls der Cluster bereits läuft, kann `k3d cluster start teko-k8s` übersprungen werden.

Danach die beiden Port-Forwards erneut starten:

```powershell
kubectl -n kube-system port-forward service/traefik 8081:80
```

```powershell
kubectl -n food-delivery port-forward service/rabbitmq 15672:15672
```

## 2.4. Umgebung beenden

Die Port-Forwards in den jeweiligen Terminals mit `Ctrl + C` beenden.

Cluster stoppen, ohne ihn zu löschen:

```powershell
k3d cluster stop teko-k8s
```

# 3. Was wurde umgesetzt?

## 3.1. Block 03 – Deployments, Services und ConfigMaps

- Control API und Dashboard als Container-Images gebaut
- Images in den k3d-Cluster importiert
- Kubernetes-Manifeste mit Kustomize angewendet
- Namespace `food-delivery` verwendet
- Deployments, Services und EndpointSlices geprüft
- Kubernetes-DNS getestet
- ConfigMap `simulation-config` verwendet
- Readiness- und Liveness-Probes geprüft
- Konfigurationsänderung per Rollout übernommen

## 3.2. Block 04 – Ingress und externe Zugriffe

- Ingress `food-delivery` mit Ingress-Klasse `traefik` eingerichtet
- Dashboard auf zwei Replikas skaliert
- gemeinsamer HTTP-Einstieg für Dashboard, API, Health und Metrics eingerichtet
- Load Balancing zwischen zwei Dashboard-Pods sichtbar gemacht
- Self-Healing durch Löschen eines Dashboard-Pods geprüft

## 3.3. Block 05 – Messaging mit RabbitMQ

- Betriebsmodus auf `APP_MODE=distributed` umgestellt
- RabbitMQ `4.3.5-management-alpine` als StatefulSet eingesetzt
- persistenter Speicher über `data-rabbitmq-0` mit `1 GiB` verwendet
- RabbitMQ-Service für AMQP auf Port `5672` und Management auf Port `15672` bereitgestellt
- Customer Simulator und Courier Simulator als StatefulSets betrieben
- Order Worker mit zwei Replikas betrieben
- Restaurant Worker für Pizza, Bowl und Curry bereitgestellt
- Nachrichtenrückstau in der Pizza-Queue erzeugt
- Rückstau mit zwei Competing Consumers abgebaut
- RabbitMQ-Probes für die lokale Umgebung angepasst

## 3.4. Architektur nach Block 05

```text
Browser / Client
       |
       v
Traefik Ingress
       |
       +-- /         -> Dashboard
       +-- /api      -> Control API
       +-- /health   -> Control API
       +-- /metrics  -> Control API

Producer und Worker
       |
       v
RabbitMQ
       |
       +-- Restaurant Queues
       +-- Order Projection
       +-- Courier Dispatch
       +-- Dead Letter Queue
```

RabbitMQ wird im Dashboard gelb dargestellt, weil die Darstellung den Message Broker als eigene Komponentenkategorie kennzeichnet. Der technische Zustand wird durch die Anzeige `1/1` bestimmt.

# 4. Technische Nachweise

## 4.1. Block 03

```powershell
kubectl kustomize ./deploy/overlays/block-03-standalone
kubectl -n food-delivery get svc,endpointslice
kubectl -n food-delivery get pods --show-labels
```

DNS-Test:

```powershell
kubectl run dns-test --image=curlimages/curl:8.8.0 -n food-delivery --restart=Never --rm -i -- curl -fsS http://control-api:8080/health/ready
```

## 4.2. Block 04

```powershell
kubectl kustomize ./deploy/overlays/block-04-ingress
kubectl -n food-delivery get deployment,ingress
```

Routen testen:

```powershell
(Invoke-WebRequest -UseBasicParsing -Uri "http://127.0.0.1:8081/").StatusCode
(Invoke-WebRequest -UseBasicParsing -Uri "http://127.0.0.1:8081/api/v1/snapshot").StatusCode
(Invoke-WebRequest -UseBasicParsing -Uri "http://127.0.0.1:8081/health/ready").StatusCode
```

Erwartet:

```text
200
200
200
```

## 4.3. Block 05

Gesamtzustand:

```powershell
kubectl -n food-delivery get deploy,statefulset,pods,pvc
kubectl -n food-delivery exec rabbitmq-0 -- rabbitmqctl list_queues name consumers messages_ready
```

### 4.3.1. Rückstau und Competing Consumers

Pizza-Consumer stoppen und acht Events publizieren:

```powershell
kubectl -n food-delivery scale deployment/restaurant-pizza --replicas=0
kubectl -n food-delivery wait --for=delete pod -l app.kubernetes.io/instance=restaurant-pizza --timeout=120s
.\scripts\lab-publish.ps1 -Mode valid -Count 8
kubectl -n food-delivery exec rabbitmq-0 -- rabbitmqctl list_queues name consumers messages_ready
```

Zwei Consumer starten:

```powershell
kubectl -n food-delivery scale deployment/restaurant-pizza --replicas=2
kubectl -n food-delivery rollout status deployment/restaurant-pizza --timeout=180s
kubectl -n food-delivery exec rabbitmq-0 -- rabbitmqctl list_queues name consumers messages_ready
kubectl -n food-delivery logs -l app.kubernetes.io/instance=restaurant-pizza --prefix --since=2m
```

Beobachtetes Resultat:

```text
Vorher:  0 Consumer, 12 bis 13 messages_ready
Nachher: 2 Consumer, 0 messages_ready

Pod 1: restaurant-pizza-6b7d8c5486-5wgbn
Pod 2: restaurant-pizza-6b7d8c5486-sggkg
```

Der Wert lag über acht Nachrichten, weil der laufende Customer Simulator während des Tests zusätzliche Pizza-Bestellungen publizierte.

Nach dem Test auf den Normalzustand zurückskalieren:

```powershell
kubectl -n food-delivery scale deployment/restaurant-pizza --replicas=1
kubectl -n food-delivery rollout status deployment/restaurant-pizza --timeout=180s
```

# 5. Fehlerbehebung

## 5.1. RabbitMQ bleibt `0/1` oder startet wiederholt neu

Status und Events prüfen:

```powershell
kubectl -n food-delivery get pod rabbitmq-0 -o wide
kubectl -n food-delivery get pvc
kubectl -n food-delivery describe pod rabbitmq-0
kubectl -n food-delivery logs rabbitmq-0
```

In der lokalen Umgebung mussten die Probe-Timeouts auf fünf Sekunden erhöht werden. Der erfolgreiche Zustand lautet:

```text
rabbitmq-0   1/1   Running   0
```

## 5.2. Port bereits belegt

Einen anderen freien lokalen Port verwenden, zum Beispiel:

```powershell
kubectl -n kube-system port-forward service/traefik 8080:80
```

## 5.3. ImagePullBackOff

Images erneut bauen und in `teko-k8s` importieren:

```powershell
.\scripts\build-images.ps1
.\scripts\load-images.ps1 -Cluster teko-k8s
```

## 5.4. Diagnose

```powershell
k3d cluster list
kubectl config current-context
kubectl get nodes -o wide
kubectl -n food-delivery get deploy,statefulset,pods,svc,endpointslice,ingress,cm,pvc -o wide
kubectl -n food-delivery get events --sort-by='.lastTimestamp'
```

# 6. Git-Workflow

```powershell
git pull --rebase origin main
git status
git add .
git commit -m "Add Block 05 messaging with RabbitMQ"
git push origin main
```

# 7. Aufräumen

Nur die Dispatch-City-Ressourcen entfernen:

```powershell
kubectl delete namespace food-delivery
```

Cluster vollständig löschen:

```powershell
k3d cluster delete teko-k8s
```
