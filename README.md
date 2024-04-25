## GitOps
- Ansatz in der Softwareentwicklung und -bereitstellung
- Ziel: Prinzipien der Git-Versionierungskontrollen auf die IT-Infrastruktur und den Betrieb (Operations) anwenden
- GitOps = Git + Operations
- Git als Single Source of Truth für die Deklaration und Verwaltung von Infrastruktur und Anwendungen
- sämtliche Konfigurationen und Codeänderungen werden zuerst in Git als Repository eingefügt und versioniert, bevor sie wirklich in die Produktionsumgebung übernommen werden
#### Vorteile
- Automatisierung: Continous Deployment hilft dabei, Änderungen automatisch zu testen und in die Produktionsumgebung einzuspielen, sobald sie im Git-Repository gespeichert sind
- Transparenz, Nachvollziehbarkeit: Jede Änderung wird im Git-Repository gespeichert und kann nachvollzogen werden. Dadurch kann es überprüft werden wer, wann, was geändert hat.
- Schnellere Fehlerbehebung: Alle Änderungen werden über Pull Requests verwaltet. Dadurch können Fehler schneller identifiziert und behoben bzw. rückgängig gemacht werden.

#### GitOps Prinzipien
Während herkömmliche Pipelines und die darin enthaltenen Tools wie GitHub Actions oder GitLab CI/CD immer dann ausgeführt werden, wenn ein bestimmtes Ereignis wie ein Git-Push eintritt, funktioniert das GitOps-Prinzip genau andersherum. 
Der größte Unterschied besteht darin, dass der gewünschte Zustand kontinuierlich aus einer Quelle wie einem Github-Repository gezogen wird, um die Infrastruktur zu steuern. Dieses Muster wird als Reconciliation Loop bezeichnet und entspricht dem Kubernetes-Operator-Muster.
Die Elemente von GitOps können durch vier Prinzipien beschrieben werden.
1. **Deklarative Konfiguration**: Die gesamte Infrastruktur wird in einer deklarativen Konfiguration beschrieben, die in einem Git-Repository gespeichert ist.
2. **Versionierung**: Alle Änderungen an der Infrastruktur werden in einem Git-Repository versioniert.
3. **Automatisierung**: Die Infrastruktur wird automatisch auf den gewünschten Zustand gebracht, sobald eine Änderung im Git-Repository festgestellt wird.
4. **Beobachtung**: Der Zustand der Infrastruktur wird kontinuierlich überwacht und bei Abweichungen automatisch korrigiert.

## ArgoCD
- spezifisches Tool, das innerhalb des GitOps-Paradigmas verwendet wird
- deklaratives, Git-basiertes Continous Deployment-Tool für Kubernetes
- ArgoCD automatisiert den Deployment-Prozess, indem es die Konfigurationen in einem Git-Repository überwacht und die Änderungen in der Kubernetes-Clusterumgebung anwendet
- ArgoCD überwacht das Git-Repository und synchronisiert die Konfigurationen mit dem Kubernetes-Cluster: Wenn eine Änderung in der Konfiguration vorgenommen wird, wird diese Änderung automatisch auf den Cluster angewendet

#### Funktionsweise
- Deklarative Konfigurationen in einem Git-Repository speichern: Konfiguration der Anwendung wird in YAML oder JSON-Dateien (auch Helm Charts oder Kustomize möglich) gespeichert und in einem Git-Repository abgelegt
- Automatische Synchronisation mit dem Kubernetes-Cluster: ArgoCD überwacht das Git-Repository und synchronisiert die Konfigurationen mit dem Kubernetes-Cluster
- Self-Healing: Wenn die tatsächlichen Zustände im Cluster von den deklarierten Zuständen im Git abweichen, kann ArgoCD diese automatisch korrigieren. Dadurch wird der Cluster in den gewünschten Zustand versetzt
- Benutzerfreundliches Dashboard: ArgoCD bietet ein Dashboard, das die Anwendungen und deren Zustände im Cluster anzeigt

### Tutorial
Wir gehen nach folgender Anleitung vor:
https://argo-cd.readthedocs.io/en/stable/getting_started/
1. Installation von ArgoCD
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Dadurch wird ein neuer Namespace `argocd` erstellt, in dem die Argo-CD-Dienste installiert werden. Checken kannst du das mit `kubectl get pods -n argocd`.
2. Download des CLI-Tools von ArgoCD CLI
```
brew install argocd
```
```
choco install argocd-cli
```

3. Zugriff auf den Argo CD API Server
Standardmäßig ist der Argo CD API-Server nicht mit einer externen IP-Adresse versehen. Um auf den API-Server zuzugreifen, wähle eine der folgenden Techniken, um den Argo CD API-Server freizugeben:
- Port-Weiterleitung
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
- Service-Typ LoadBalancer
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
- Ingress
Es gibt eine ingress Dokumentation (https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/) für ArgoCD, die du verwenden kannst, um den Zugriff auf den API-Server zu konfigurieren.

4. Login in die CLI
Das initiale Passwort für den `admin`-Benutzer wird automatisch generiert und als Klartext im Feld `password` in einem Secret mit dem Namen `argocd-initial-admin-secret` im `argocd`-Namespace gespeichert. Um das Passwort zu erhalten, führe den folgenden Befehl aus:
```
argocd admin initial-password -n argocd
```
Das Passwort wird dann angezeigt.
Danach kannst du den `127.0.0.1:<Port-Forwarding-Nummer>` in deinem Browser öffnen und dich mit dem Benutzernamen `admin` und dem Passwort einloggen.
Alternativ bitte per CLI einloggen:
```
argocd login 127.0.0.1:<Port-Forwarding-Nummer> --insecure
```


5. Erstellen eines Projekts
Wir gehen mal das folgende Beispiel durch: https://github.com/argoproj/argocd-example-apps
Zuerst müssen wir den aktuellen Namespace auf argocd setzen, indem wir den folgenden Befehl ausführen:
```
kubectl config set-context --current --namespace=argocd
```
Erstelle dann die Beispiel-Gästebuchanwendung mit dem folgenden Befehl:
``` 
argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default --server 127.0.0.1:<Port-Forwaring-Nummer> --insecure
```

6. Synchronisieren der Anwendung
Sobald die Gästebuchanwendung erstellt ist, können wir nun ihren Status einsehen:
```
➜  ~ argocd app get guestbook
Name:               argocd/guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://127.0.0.1:8080/applications/guestbook
Repo:               https://github.com/argoproj/argocd-example-apps.git
Target:
Path:               guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        OutOfSync from  (d7927a2)
Health Status:      Missing

GROUP  KIND        NAMESPACE  NAME          STATUS     HEALTH   HOOK  MESSAGE
       Service     default    guestbook-ui  OutOfSync  Missing
apps   Deployment  default    guestbook-ui  OutOfSync  Missing
```
Der Anwendungsstatus befindet sich zunächst im Zustand OutOfSync, da die Anwendung noch nicht bereitgestellt wurde und keine Kubernetes-Ressourcen erstellt worden sind. Um die Anwendung zu synchronisieren (bereitzustellen), führen wir aus:
Um die Anwendung zu synchronisieren, führe den folgenden Befehl aus:
```
argocd app sync guestbook
```
Dieser Befehl ruft die Manifeste aus dem Repository ab und führt ein kubectl apply der Manifeste durch. Die Gästebuch-App läuft nun und Sie können ihre Ressourcenkomponenten, Protokolle, Ereignisse und den bewerteten Status anzeigen.

7. Eigene Beispiel-Anwendung durchstarten
Wir haben bereits ein Repository angelegt mit einer Beispielanwendung, die aus einem NestJS-Backend und einem Angular-Frontend besteht. Dieses Repository fügen wir einfach ArgoCD wieder hinzu mit folgenden Befehl:
```
argocd app create myapp --repo https://github.com/marcolindner/example-k8s-application --path ./deployment --dest-server https://kubernetes.default.svc --dest-namespace default --server 127.0.0.1:8080 --insecure
```
Danach können wir die Anwendung synchronisieren:
```
argocd app sync myapp
```
Das soltle nun funktionieren. 
