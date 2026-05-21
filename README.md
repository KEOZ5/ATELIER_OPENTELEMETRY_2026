# OpenTelemetry — Atelier 2026

> **OpenTelemetry** (« OTel ») est le standard ouvert de référence pour la **métrologie**, la **traçabilité** et la **supervision** des systèmes informatiques modernes.
> Il unifie la collecte des **logs**, **métriques** et **traces** d'applications distribuées et les exporte vers des outils comme Prometheus, Grafana, Jaeger, Zipkin, Datadog, New Relic ou Splunk.

---

## Sommaire

1. [Architecture cible](#architecture-cible)
2. [Prérequis](#prérequis)
3. [Étape A — Jaeger](#étape-a--jaeger)
4. [Étape B — OpenTelemetry Collector](#étape-b--opentelemetry-collector)
5. [Étape C — Application Flask](#étape-c--application-flask)
6. [Synthèse](#synthèse)
7. [Composants d'OpenTelemetry](#composants-dopentelemetry)

---

## Architecture cible

<img width="1661" height="947" alt="ChatGPT Image 21 mai 2026, 14_44_58" src="https://github.com/user-attachments/assets/7c37163e-6c45-481b-8e21-063aa4d8aedc" />


> OpenTelemetry est au monitoring ce que TCP/IP est au réseau : un langage commun, neutre et extensible pour observer tout type de système.

---

## Prérequis

- Un **Codespace GitHub** ouvert sur ce dépôt
- **Docker** disponible dans le terminal (présent par défaut)
- **Python 3** (présent par défaut)

---

## Étape A — Jaeger

**Jaeger** est l'outil de visualisation des traces. Il affiche graphiquement les traces collectées par OpenTelemetry.

### 1. Lancer Jaeger

Copier-coller dans le terminal :

```bash
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 14318:4318 \
  -p 4317:4317 \
  jaegertracing/all-in-one:1.53
```

### 2. Exposer les ports

| Port    | Usage                            | Visibilité |
| ------- | -------------------------------- | ---------- |
| `16686` | Interface Web Jaeger             | **Public** |
| `14318` | API OTLP/HTTP (entrée Jaeger)    | **Public** |

> Dans l'onglet **PORTS** du Codespace : *clic droit sur le port → Visibilité → Public*.

### 3. Récupérer l'URL de l'API Jaeger

Dans l'onglet **PORTS**, copier l'URL associée au port **14318**. Exemple :

```
https://congenial-train-4vp4vgqppv36wr-14318.app.github.dev/
```

> Un message *« 404 page not found »* à cette URL est **normal** : il s'agit d'une API, pas d'une interface Web.

Conserver cette URL — elle sera déclarée comme endpoint d'export dans la configuration d'OpenTelemetry.

---

## Étape B — OpenTelemetry Collector

Le **Collector** est un *concentrateur* de traces (et de logs / métriques). Il reçoit les données émises par les applications, les transforme et les transmet à un backend — ici Jaeger.

### 1. Créer le fichier de configuration

```bash
nano otel-collector-config.yaml
```

```yaml
receivers:
  otlp:
    protocols:
      http:                       # écoute sur 4318/tcp
        endpoint: 0.0.0.0:4318

processors:
  batch: {}

exporters:
  otlphttp:
    endpoint: https://congenial-train-4vp4vgqppv36wr-14318.app.github.dev   # URL vers Jaeger port 4318
  logging:
    loglevel: info

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp, logging]
```

> **Important** : le champ `endpoint` doit être **votre URL Jaeger** (celle du port 14318), pas celle de l'exemple.

Enregistrer : `Ctrl + S` puis `Ctrl + X`.

### 2. Lancer le Collector

```bash
docker run -d --name otel-collector \
  -p 4318:4318 \
  -v $PWD/otel-collector-config.yaml:/etc/otelcol/config.yaml:ro \
  otel/opentelemetry-collector-contrib:0.101.0 \
  --config=/etc/otelcol/config.yaml
```

Le Collector reçoit les spans des applications sur **4318** et les transmet à Jaeger sur **14318**.

### 3. Récupérer l'URL de l'API Collector

Dans l'onglet **PORTS**, copier l'URL associée au port **4318**. Exemple :

```
https://congenial-train-4vp4vgqppv36wr-4318.app.github.dev/
```

> Penser à passer ce port en **Public** également.

Cette URL sera l'endpoint déclaré dans l'application Flask.

---

## Étape C — Application Flask

L'application envoie ses traces au Collector OpenTelemetry.

### 1. Créer le fichier `app.py`

```bash
nano app.py
```

```python
#!/usr/bin/env python3
from flask import Flask, jsonify
import time, random

app = Flask(__name__)

@app.route("/")
def index():
    return "Hello OTel from Flask!"

@app.route("/work")
def work():
    time.sleep(random.uniform(0.05, 0.25))
    return jsonify(status="ok")

@app.route("/error")
def boom():
    raise RuntimeError("boom")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Enregistrer : `Ctrl + S` puis `Ctrl + X`.

### 2. Installer l'environnement Python

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install --upgrade pip
pip install flask \
            opentelemetry-distro \
            opentelemetry-exporter-otlp \
            opentelemetry-instrumentation-flask
```

### 3. Configurer les variables d'environnement

> **Important** : `OTEL_EXPORTER_OTLP_ENDPOINT` doit pointer vers **l'URL 4318 de votre Collector**.

```bash
export OTEL_SERVICE_NAME="demo-app"
export OTEL_RESOURCE_ATTRIBUTES="deployment.environment=lab,service.version=1.0.0"
export OTEL_TRACES_SAMPLER="always_on"
export OTEL_EXPORTER_OTLP_ENDPOINT="https://congenial-train-4vp4vgqppv36wr-4318.app.github.dev"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
```

### 4. Lancer l'application instrumentée

```bash
opentelemetry-instrument \
  --traces_exporter otlp \
  --metrics_exporter none \
  python app.py
```

### 5. Générer des traces

Ouvrir l'application via l'onglet **PORTS** (port **5000**), puis appeler :

| Route              | Comportement                          |
| ------------------ | ------------------------------------- |
| `/`                | Page d'accueil                        |
| `/work`            | Requête nominale (latence simulée)    |
| `/error`           | Requête en erreur (exception levée)   |

Les traces apparaissent dans Jaeger sous le service **`demo-app`**.

### Travail demandé

> Fournir une **copie d'écran** de l'interface Jaeger affichant les traces générées.

---

## Synthèse

OpenTelemetry est un ensemble d'outils et de bibliothèques qui permettent de **collecter**, **transformer** et **exporter** les trois piliers de l'observabilité :

| Pilier         | Définition                                                | Exemple                                                |
| -------------- | --------------------------------------------------------- | ------------------------------------------------------ |
| **Traces**     | Déroulement d'une requête de bout en bout du système      | Une requête API a mis 248 ms du frontend au backend    |
| **Métriques**  | Mesures numériques périodiques                            | Taux d'erreur, latence moyenne, CPU, mémoire           |
| **Logs**       | Messages texte structurés                                 | `Erreur 500 sur /work à 14:32:01`                      |

OpenTelemetry sert de **colle universelle** entre vos applications et les outils de supervision (Grafana, Prometheus, Jaeger, Datadog, etc.).

---

## Composants d'OpenTelemetry

### Instrumentation

Partie intégrée à l'application (ici Flask) qui :

- mesure le comportement (temps de réponse, erreurs, etc.),
- crée des **spans** — unités élémentaires de trace,
- les envoie au Collector via le protocole **OTLP**.

### Collector

Proxy d'observabilité, en charge de :

- **recevoir** les données des applications (port 4318),
- **traiter** : batch, sampling, anonymisation, enrichissement,
- **exporter** vers un backend (ici Jaeger).

### Backend (Jaeger)

Dashboard d'observation, qui :

- **stocke** les traces dans une base intégrée (Badger / Elasticsearch),
- fournit une **interface graphique** sur le port `16686` pour suivre le parcours d'une requête à travers plusieurs services,
- structure les données ainsi : *une requête = une trace = plusieurs spans liés*.
