# Monitoring Docker Swarm – 
Prometheus + Grafana + cAdvisor + Node Exporter + Swarm Scheduler Exporter

Ce projet met en place une stack de monitoring complète pour un **cluster Docker Swarm** de 4 nœuds (1 manager + 3 workers).
On garde une machine en standalone (séparée du cluster) afin d'héberger Prometheus et Grafana (services critiques à ne pas orchestrer).
Suite logique de https://github.com/BangSkyline/Pluton-Charon -> Monitoring d'une instance Docker Standalone.

**Composants principaux :**

- Prometheus (scraping et stockage des métriques)

- Grafana (visualisation et dashboards)

- Node Exporter (métriques système des nœuds)

- cAdvisor (métriques conteneurs Docker)

- Swarm Scheduler Exporter (métriques niveau Swarm : services, tâches, scheduling, etc.)

## Architecture

- **Prometheus** : tourne en standalone sur IP fixe 192.168.95.40:8180 → 9090

- **Grafana** : tourne en standalone sur IP fixe 192.168.95.42:8184 → 3000

- **Node Exporter** : mode global sur tous les nœuds (9100)

- **cAdvisor** : mode global sur tous les nœuds (8183)

- **Swarm Scheduler Exporter** : replicated, 1 replica, contraint sur le manager (9338)

Réseaux :

- `cosmoserv1` → réseau macvlan pour Prometheus + Grafana (standalone)

- `cosmos` → réseau overlay Swarm pour les exporters

## Prérequis

- Docker et Docker Swarm installés et opérationnels

- Un cluster Swarm avec 4 nœuds (IPs : 192.168.95.10 à .13)

- Réseau overlay nommé `cosmos` déjà créé

- Réseau MacVLAN nommé `cosmoserv1` déjà créé

- Accès root sur le manager/workers et sur la machine standalone

## Installation

### 1. Création des répertoires et fichier de configuration Prometheus

Sur la machine où tournera Prometheus (standalone) :

```bash

sudo mkdir -p /opt/monitoring/prometheus/

sudo nano /opt/monitoring/prometheus/prometheus.yml

```
Prometheus.yml : 

```bash

global:
  scrape_interval:     15s
  evaluation_interval: 15s
 
scrape_configs:

  - job_name: "prometheus"
    static_configs:
      - targets: ["192.168.95.40:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets:
          - "192.168.95.10:9100"
          - "192.168.95.11:9100"
          - "192.168.95.12:9100"
          - "192.168.95.13:9100"
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: instance
        replacement: '${1}'

  - job_name: "cadvisor"
    static_configs:
      - targets:
          - "192.168.95.10:8183"
          - "192.168.95.11:8183"
          - "192.168.95.12:8183"
          - "192.168.95.13:8183"
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: instance
        replacement: '${1}'
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: host
        replacement: '${1}'

  - job_name: "swarm-scheduler-exporter"
    metrics_path: /metrics
    scheme: http
    static_configs:
      - targets:
          - "192.168.95.10:9323"
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):.*'
        target_label: instance
        replacement: '$1'


```

Vérification :

Bash```

ls -l /opt/monitoring/prometheus/prometheus.yml

```

(doit afficher : -rw-r--r-- 1 root root ... prometheus.yml)

### 2. Déploiement de la stack standalone (Prometheus + Grafana)

Créez le fichier docker-compose.yml sur la machine standalone.

Dans le répertoire du fichier,lancez :

```bash

docker compose up -d

```

Accès Grafana : http://192.168.95.42:8184

Identifiants : admin / Skyline

### 3. Déploiement des exporters sur le cluster Swarm (Manager)

Sur le manager Swarm :

```bash

docker stack deploy -c docker-compose-monitoring.yml monitoring

Contenu recommandé du fichier docker-compose-monitoring.yml :

```bash
version: "3.8"

networks:
  cosmos:
    external: true

services:
  node_exporter:
    image: prom/node-exporter:v1.8.1
    networks:
      - cosmos
    deploy:
      mode: global
    ports:
      - target: 9100
        published: 9100
        protocol: tcp
        mode: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|run|var/lib/docker/.+|var/lib/kubelet/.+)($$|/)"
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    networks:
      - cosmos
    deploy:
      mode: global
    ports:
      - target: 8080
        published: 8183
        protocol: tcp
        mode: host
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk:/dev/disk:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    privileged: true
    restart: unless-stopped

  swarm-exporter:
    image: ghcr.io/leinardi/swarm-scheduler-exporter:latest
    user: root
    command:
      - "-log-level=info" 
      - "-log-format=text"
      - "-containers"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - cosmos
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
    ports:
      - target: 8888                
        published: 9338
        protocol: tcp
        mode: host
    restart: unless-stopped
    
```

### Vérifications rapides :

Prometheus targets : http://192.168.95.40:8180/targets

→ Tous les jobs doivent être UP

Métriques swarm :

```bash

curl http://192.168.95.10:9338/metrics | grep swarm

```

Grafana : ajoutez la datasource Prometheus

URL : http://192.168.95.40:8180

## Améliorations futures possibles :

Ajout d'alerting Grafana (OOM, CPU throttle, disque >90%, replicas manquants)

Provisioning automatique des dashboards via YAML

Intégration Loki + Promtail pour les logs

Blackbox exporter pour surveiller les endpoints applicatifs

Mise en place de Node Exporter textfile collector pour métriques custom

