# EK951 Verteilte Dateisysteme "Shared Storage" (BORM)
## Einführung 
### AI Data Workflows mit Kafka und MinIO

Dieser Workflow demonstriert die Implementierung eines ereignisgesteuerten Datenpipeline-Systems unter Verwendung von MinIO und Apache Kafka. Der Hauptzweck ist die automatische Verarbeitung von Bildern, sobald sie in ein MinIO-Bucket hochgeladen werden.

### Überblick

Der Workflow besteht aus folgenden Komponenten:

1. **MinIO als Produzent**: Speichert eingehende Rohdaten (Bilder) und sendet Ereignisbenachrichtigungen an Kafka.
2. **Apache Kafka als Broker**: Verwaltet die Nachrichtenwarteschlange und stellt Nachrichten für Konsumenten bereit.
3. **MinIO als Konsument**: Verarbeitet die Nachrichten aus Kafka, führt Bildverarbeitungsoperationen durch und speichert die Ergebnisse.

### Technologien

- **MinIO**: Objekt-Storage-System für die Speicherung von Roh- und verarbeiteten Bildern.
- **Apache Kafka**: Verteiltes Messaging-System für die Ereignisverarbeitung.
- **Kubernetes**: Container-Orchestrierungsplattform für die Bereitstellung der Dienste.
- **Python**: Programmiersprache für die Implementierung des Konsumenten-Skripts.

### Funktionsweise

1. Bilder werden in ein MinIO-Bucket hochgeladen.
2. MinIO sendet Ereignisbenachrichtigungen an ein Kafka-Topic.
3. Ein Python-Skript konsumiert die Kafka-Nachrichten.
4. Das Skript lädt die Bilder aus MinIO, verarbeitet sie (z.B. Größenänderung) und lädt sie in ein Ziel-Bucket hoch.

Dieser Workflow demonstriert die Leistungsfähigkeit von MinIO und Kafka für die Erstellung skalierbarer, ereignisgesteuerter Datenpipelines in einer Kubernetes-Umgebung.

## Umsetzung
### 1. Install kubectl on macOS 
You must use a kubectl version that is within one minor version difference of your cluster. For example, a v1.32 client can communicate with v1.31, v1.32, and v1.33 control planes. Using the latest compatible version of kubectl helps avoid unforeseen issues.

Run the installation command:

`brew install kubectl` or `brew install kubernetes-cli`

Test to ensure the version you installed is up-to-date: `kubectl version --client`

### 2. Install `kind`
kind is a tool for running local Kubernetes clusters using Docker container “nodes”.
kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

There are downloadable release binaries, community-managed packages, and a source installation guide.

On macOS:
```
# For Intel Macs
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.26.0/kind-darwin-amd64
# For M1 / ARM Macs
[ $(uname -m) = arm64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.26.0/kind-darwin-arm64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```

### 3. Create and configure kind cluster 
Creating a Kubernetes cluster: 
`kind create cluster`

To specify a configuration file when creating a cluster, use the `--config` flag:

`kind create cluster --config kind-config.yaml`

**1. kind-config.yaml**

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

**2. Launch the cluster (this may take several minutes)**

`kind create cluster --name minio-kafka --config kind-config.yaml`

Set kubectl context to "kind-minio-kafka"
You can now use your cluster with:

`kubectl cluster-info --context kind-minio-kafka`

**3. Verify the cluster is up**

`kubectl get no`

### 4. Installing Kafka
Kafka requires a few services to be running to support it before it can be run. These services are:
* Certmanager
* Zookeeper


**1. Install cert-manager in Kubernetes cluster**

`kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v1.6.2/cert-manager.yaml`

**2. Check the status to verify cert-manager resources have been created**
* `kubectl get ns`
* `kubectl get po -n cert-manager`

**3. Install zookeeper using Helm charts**
* install helm `brew install helm`
* add "pravega" to your repositories`helm repo add pravega https://charts.pravega.io`
* `helm repo update`
* `helm install zookeeper-operator --namespace=zookeeper --create-namespace pravega/zookeeper-operator`
* `kubectl --namespace zookeeper create -f - <<EOF

**4. Verify that both the zookeeper operator and cluster pods are running**

`kubectl -n zookeeper get po`
![](assets/17377261045893.png)

### 5. Install Kafka cluster components
Kafka has an operator called Koperator which we’ll use to manage our Kafka install. It will take about 4-5 minutes for the Kafka cluster to come up.

![](assets/17376303318928.png)

Problem: Banzaiclou d nicht erreichbar 

Lösung: 
Da Banzai Cloud's Koperator nicht erreicbbar war, haben wir stattdessen **Strimzi Kafka Operator** als alternative verwendet, um die Apache Kafka installation on Kubernetes zu managen. Strimzi is a widely adopted and actively maintained open-source project that simplifies running Apache Kafka on Kubernetes. ([GitHub](https://github.com/strimzi/strimzi-kafka-operator?utm_source=chatgpt.com))

**Installation Steps:**

1. **Add the Strimzi Helm Repository:**

   ```bash
   helm repo add strimzi https://strimzi.io/charts/
   helm repo update
   ```

2. **Install the Strimzi Kafka Operator:**

   ```bash
   helm install strimzi-kafka-operator strimzi/strimzi-kafka-operator --namespace kafka --create-namespace
   ```

3. **Verify the Installation:**

   ```bash
   kubectl get pods -n kafka
   ```

   Ensure that the Strimzi operator pods are running successfully.

**Additional Resources:**

- **Strimzi Documentation:** For detailed guidance on configuring and managing your Kafka clusters with Strimzi, refer to the official documentation. ([GitHub](https://github.com/strimzi/strimzi-kafka-operator?utm_source=chatgpt.com))

By adopting Strimzi, you can effectively manage your Kafka clusters on Kubernetes, benefiting from its robust features and active community support. 

Run `kubectl -n kafka get po` to confirm that Kafka has started. It takes a few minutes for Kafka to be operational. Please wait before proceeding.

### 6. Configuring Kafka topic

Create a topic called my-topic`

```
kubectl apply -n kafka -f - <<EOF
apiVersion: kafka.banzaicloud.io/v1alpha1
kind: KafkaTopic
metadata:
    name: my-topic
spec:
    clusterRef:
        name: kafka
    name: my-topic
    partitions: 1
    replicationFactor: 1
    config:
        "retention.ms": "604800000"
        "cleanup.policy": "delete"
EOF
```


### 7. Installation von MinIO in Kubernetes

#### Problem

Nach der Installation von MinIO gemäß den angegebenen Schritten:

1. **Repository klonen**:
   ```bash
   git clone https://github.com/minio/operator.git
   ```

2. **Ressourcen anwenden**:
   ```bash
   kubectl apply -k operator/resources
   ```

3. **Tenant-Lite anwenden**:
   ```bash
   kubectl apply -k operator/examples/kustomization/tenant-lite
   ```

4. **MinIO-Konsole überprüfen**:
   ```bash
   kubectl -n tenant-lite get svc | grep -i console
   ```

trat ein Problem auf: Der letzte Befehl funktionierte nicht, da keine Ressourcen im `tenant-lite`-Namespace vorhanden waren.

#### Fehlermeldung
```bash
kubectl -n tenant-lite get svc | grep -i console
No resources found in tenant-lite namespace.
```

### 7. Lösungsvorschlag des Professors

Nach Rücksprache mit unserem Professor stellte sich heraus, dass ein Konfigurationsfile abgeändert werden muss. Das Ziel war, die Konfiguration korrekt anzupassen, um die Ressourcenerstellung im `tenant-lite`-Namespace zu ermöglichen. 

#### Unser Ansatz

Nachdem wir den Aufbau des Konfigurationsfiles besser verstanden hatten, haben wir versucht, die notwendigen Änderungen vorzunehmen. Leider scheiterten wir an der korrekten Anpassung, da die Abhängigkeiten und Anforderungen der MinIO-Konfiguration nicht vollständig klar waren.

Wir planen, weitere Unterstützung einzuholen oder eine alternative Lösung zu finden, um MinIO erfolgreich im Kubernetes-Cluster zu starten und die Konsole zugänglich zu machen.


## Apache Kafka: Einführung, Architektur und Einsatzgebiete

#### **Was ist Apache Kafka?**

Apache Kafka ist eine Open-Source-Plattform zur Verarbeitung von Datenströmen in Echtzeit. Es wurde ursprünglich von LinkedIn entwickelt und später als Teil der Apache Software Foundation bereitgestellt. Kafka ist darauf ausgelegt, große Mengen an Daten effizient und mit geringer Latenz zu verarbeiten, zu speichern und weiterzuleiten. Es wird häufig als verteilter Event-Streaming-Dienst oder Message-Broker bezeichnet.

---

#### **Funktionsweise und Architektur von Apache Kafka**

Die Architektur von Kafka basiert auf drei zentralen Komponenten: **Produzenten**, **Broker** und **Konsumenten**. Zusätzlich spielt das Konzept von **Themen** und **Partitionen** eine entscheidende Rolle:

1. **Produzenten (Producers)**  
   - Produzenten sind Anwendungen oder Dienste, die Daten an Kafka senden. Sie veröffentlichen Nachrichten in sogenannte **Themen** (Topics), die als logische Kategorien für Daten dienen.
   
2. **Broker**  
   - Die Broker sind die Server, die die Kernfunktion von Kafka ausführen. Sie speichern die Daten und machen sie für Konsumenten zugänglich. Kafka ist als verteiltes System aufgebaut, wobei mehrere Broker zusammenarbeiten, um Skalierbarkeit und Ausfallsicherheit zu gewährleisten.

3. **Konsumenten (Consumers)**  
   - Konsumenten sind Anwendungen, die Daten aus Kafka abrufen. Sie abonnieren bestimmte Themen und lesen die Nachrichten in der Reihenfolge, in der sie veröffentlicht wurden.

4. **Themen und Partitionen**  
   - Ein **Thema** ist eine Kategorie oder ein Feed-Name, unter dem Nachrichten veröffentlicht werden.  
   - Jedes Thema ist in **Partitionen** aufgeteilt. Partitionen ermöglichen die parallele Verarbeitung und Speicherung von Daten, indem Nachrichten innerhalb eines Themas auf mehrere Broker verteilt werden.  
   - Jede Nachricht innerhalb einer Partition hat eine eindeutige Offset-ID, die ihre Position definiert.

5. **Replikation**  
   - Um Ausfallsicherheit zu gewährleisten, repliziert Kafka Partitionen über mehrere Broker hinweg. Eine Partition hat immer einen führenden Broker (Leader), der Lese- und Schreibanfragen verarbeitet, während die anderen Broker als Replikate dienen.

---

#### **Einsatzgebiete von Apache Kafka**

Kafka wird in einer Vielzahl von Szenarien eingesetzt, insbesondere in Systemen, die große Datenmengen in Echtzeit verarbeiten müssen:

1. **Echtzeit-Datenverarbeitung**  
   - Kafka wird häufig verwendet, um Streaming-Daten in Echtzeit zu verarbeiten. Beispiele sind Klickströme auf Websites, Transaktionsdaten oder IoT-Daten.

2. **Datenintegration**  
   - Unternehmen nutzen Kafka, um Daten zwischen verschiedenen Systemen und Anwendungen auszutauschen. Es dient als zentrales Backbone, um Systeme wie Datenbanken, Analytik-Tools und APIs zu verbinden.

3. **Protokollierung und Monitoring**  
   - Kafka kann als zentrales System für die Sammlung und Speicherung von Logdaten dienen. Dies ist nützlich für Fehlerbehebung, Sicherheitsüberwachung und Systemanalysen.

4. **Microservices-Kommunikation**  
   - In Microservices-Architekturen wird Kafka oft als Message-Broker verwendet, um asynchrone Kommunikation zwischen Diensten zu ermöglichen.

5. **Event-Sourcing**  
   - Kafka eignet sich hervorragend für Event-Sourcing-Ansätze, bei denen der Zustand eines Systems durch eine Reihe von Ereignissen definiert wird.

---

#### **Beispielhafte Anwendung**

Ein Online-Shop könnte Kafka nutzen, um Bestellungen in Echtzeit zu verarbeiten:  

1. **Produzent**: Das Bestellsystem sendet Bestellinformationen an ein Kafka-Thema, z. B. "Bestellungen".  
2. **Broker**: Die Kafka-Broker speichern die Bestelldaten und stellen sie verschiedenen Diensten zur Verfügung.  
3. **Konsumenten**: 
   - Ein Lagerverwaltungssystem abonniert das "Bestellungen"-Thema und aktualisiert den Lagerbestand.  
   - Ein Analysesystem abonniert das gleiche Thema, um Verkaufsberichte in Echtzeit zu erstellen.  

---

#### **Zusammenfassung für Laien**

Stellen Sie sich Apache Kafka wie ein Postsystem vor:  
- **Produzenten** sind die Absender, die Pakete (Nachrichten) verschicken.  
- **Themen** sind die Zieladressen (z. B. "Bestellungen").  
- **Broker** sind die Poststationen, die Pakete sortieren und ausliefern.  
- **Konsumenten** sind die Empfänger, die die Pakete abholen.  

Kafka sorgt dafür, dass die Pakete schnell, zuverlässig und in der richtigen Reihenfolge ankommen – auch wenn Millionen von Nachrichten unterwegs sind. Es ist ein System, das Datenbewegungen zwischen verschiedenen Systemen effizient organisiert.



## ### KIND (Kubernetes in Docker): Einführung und Erklärung  

#### **Was ist KIND?**  
KIND (Kubernetes in Docker) ist ein Tool, das Kubernetes-Cluster in Docker-Containern ausführt. Es wurde entwickelt, um Kubernetes-Cluster für Test-, Entwicklungs- oder CI/CD-Zwecke lokal bereitzustellen, ohne dass dafür eine umfangreiche Infrastruktur erforderlich ist.

---

#### **Funktionsweise und Architektur**  
1. **Docker-Container als Kubernetes-Knoten**:  
   KIND verwendet Docker-Container, um die verschiedenen Kubernetes-Komponenten (wie Master- und Worker-Knoten) zu simulieren.  
2. **Einfache Bereitstellung**:  
   Mit wenigen Befehlen kann ein lokaler Cluster erstellt werden, der dieselbe API und Funktionalität wie ein vollständiger Kubernetes-Cluster bietet.  
3. **Konfigurierbarkeit**:  
   KIND unterstützt benutzerdefinierte Konfigurationsdateien, um Cluster mit mehreren Knoten, speziellen Netzwerkeinstellungen und weiteren Parametern zu erstellen.

---

#### **Anwendungsgebiete**  
- **Entwicklung**: Ideal für lokale Tests von Kubernetes-Anwendungen.  
- **CI/CD**: Wird häufig in Build-Pipelines verwendet, um Anwendungen in Kubernetes-Clustern zu testen.  
- **Schulungen**: Einfache Möglichkeit, Kubernetes-Cluster auf einem Laptop oder PC zu simulieren.

---

#### **Zusammenfassung für Laien**  
Stellen Sie sich KIND als eine Mini-Version von Kubernetes vor, die komplett in Ihrem Laptop oder Computer läuft. Es ist wie ein Prototyp eines größeren Systems, das Ihnen erlaubt, mit Kubernetes zu experimentieren, ohne eine komplexe Infrastruktur einzurichten.

---

### Helm: Einführung und Erklärung  

#### **Was ist Helm?**  
Helm ist ein Paketmanager für Kubernetes. Es vereinfacht die Verwaltung, Bereitstellung und Aktualisierung von Kubernetes-Anwendungen, indem es sogenannte **Charts** verwendet – Sammlungen von YAML-Dateien, die die Konfiguration und Ressourcen einer Anwendung beschreiben.

---

#### **Funktionsweise und Architektur**  
1. **Helm Charts**:  
   - Ein Chart ist ein Paket, das alle Kubernetes-Ressourcen enthält, die für die Bereitstellung einer Anwendung erforderlich sind.  
   - Charts können an unterschiedliche Umgebungen angepasst werden (z. B. Entwicklungs- oder Produktionsumgebungen).  
2. **Helm Repositories**:  
   - Charts werden in Repositories gespeichert, die ähnlich wie App-Stores funktionieren. Nutzer können Charts herunterladen, installieren und aktualisieren.  
3. **Versionskontrolle**:  
   - Helm bietet eine Versionskontrolle für Anwendungen und ermöglicht Rollbacks, falls eine Aktualisierung fehlschlägt.

---

#### **Anwendungsgebiete**  
- **Automatisierung**: Vereinfachte Bereitstellung von komplexen Kubernetes-Anwendungen.  
- **Wiederholbarkeit**: Ermöglicht die gleiche Konfiguration in mehreren Umgebungen.  
- **Verwaltung**: Effiziente Updates und Rollbacks von Anwendungen.  

---

#### **Zusammenfassung für Laien**  
Helm ist wie ein App-Store für Kubernetes. Es gibt Ihnen ein vorgefertigtes Paket (Chart), das Sie herunterladen und installieren können, um eine Anwendung zu starten. Es vereinfacht das Management komplexer Anwendungen.

---

### Zookeeper: Einführung und Erklärung  

#### **Was ist ZooKeeper?**  
ZooKeeper ist ein verteiltes Koordinationssystem, das von der Apache Software Foundation entwickelt wurde. Es dient dazu, verteilte Anwendungen zu synchronisieren, Konfigurationen zu verwalten und Services zu entdecken.

---

#### **Funktionsweise und Architektur**  
1. **Hierarchischer Namespace**:  
   ZooKeeper verwendet eine baumartige Datenstruktur (ähnlich einem Dateisystem), um Daten zu speichern. Jeder Knoten (ZNode) kann Daten und Statusinformationen enthalten.  
2. **Leader-Follower-Architektur**:  
   - Ein ZooKeeper-Cluster besteht aus einem **Leader** und mehreren **Followern**.  
   - Der Leader ist für Schreiboperationen verantwortlich, während Follower die Daten replizieren und Leserequests bearbeiten.  
3. **Atomic Broadcast Protocol**:  
   - ZooKeeper verwendet ein Protokoll, um sicherzustellen, dass alle Knoten im Cluster die gleichen Daten sehen.

---

#### **Anwendungsgebiete**  
- **Verteilte Systeme**: Synchronisation und Koordination zwischen verschiedenen Komponenten, z. B. in Apache Kafka.  
- **Service Discovery**: Ermöglicht es Diensten, ihre Standorte und Zustände zu registrieren und anderen Diensten zur Verfügung zu stellen.  
- **Konfigurationsmanagement**: Verwaltung zentraler Konfigurationsdaten für verteilte Anwendungen.

---

#### **Zusammenfassung für Laien**  
ZooKeeper ist wie ein Manager in einem Teamprojekt. Er sorgt dafür, dass alle Teammitglieder (Systemkomponenten) über den gleichen Plan (Daten und Status) informiert sind und niemand aus der Reihe tanzt.


## Quellen 
* Tutorial: https://blog.min.io/complex-workflows-apache-kafka-minio/
* Kind Quick Start Tutorial: https://kind.sigs.k8s.io/docs/user/quick-start/?ref=blog.min.io#installation
* Helm docs: https://helm.sh/docs/intro/install/


