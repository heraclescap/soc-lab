# ELK Setup Guide

Installation et configuration du stack ELK sur VM Ubuntu 22.04 (192.168.126.10).  
Toutes les versions : **8.19.16**.

---

## Prérequis VM

```
OS    : Ubuntu 22.04 LTS Server
RAM   : 4096 Mo
CPU   : 2 cœurs
Disk  : 25 Go thin provisioned
Net   : VMnet8 NAT — IP statique 192.168.126.10/24
```

IP statique via `/etc/netplan/00-installer-config.yaml` :

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses: [192.168.126.10/24]
      routes:
        - to: default
          via: 192.168.126.2
      nameservers:
        addresses: [8.8.8.8]
```

```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

---

## 1. Elasticsearch

### Installation

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update && sudo apt install elasticsearch -y
```

> Le mot de passe du compte `elastic` s'affiche **une seule fois** à la fin de l'installation.
> Le noter immédiatement. Si perdu : `sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic`

### Tuning JVM heap

Fichier : `/etc/elasticsearch/jvm.options.d/heap.options`

```
-Xms1024m
-Xmx1024m
```

1024 Mo au lieu de 1500 Mo pour permettre la coexistence avec Kibana et Logstash
sur 4 Go de RAM.

### Configuration minimale

Elastic 8.x auto-configure TLS et xpack.security au premier démarrage.
**Ne pas écraser le fichier existant.** Ajouter uniquement ces lignes en tête de
`/etc/elasticsearch/elasticsearch.yml` (avant le bloc `BEGIN SECURITY AUTO CONFIGURATION`) :

```yaml
cluster.name: homelab-soc
node.name: elk-node-1
discovery.type: single-node
```

Commenter également `cluster.initial_master_nodes` dans le bloc auto-configuré.

### Démarrage

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

# Validation
curl -k -u elastic:'<MOT_DE_PASSE_ELASTIC>' https://192.168.126.10:9200
# Attendu : {"cluster_name":"homelab-soc", ...}
```

---

## 2. Kibana

### Installation

```bash
sudo apt install kibana -y
```

### Configuration

Fichier : `/etc/kibana/kibana.yml` — ajouter en bas du fichier :

```yaml
server.host: "192.168.126.10"
server.port: 5601
elasticsearch.hosts: ["https://192.168.126.10:9200"]
elasticsearch.ssl.verificationMode: none
elasticsearch.username: "kibana_system"
elasticsearch.password: "<MOT_DE_PASSE_KIBANA_SYSTEM>"

# Clés de chiffrement obligatoires pour les règles Indicator Match dans Kibana Security
# Minimum 32 caractères — générer avec : openssl rand -hex 32
xpack.encryptedSavedObjects.encryptionKey: "<CLE_32_CHARS>"
xpack.security.encryptionKey: "<CLE_32_CHARS>"
xpack.reporting.encryptionKey: "<CLE_32_CHARS>"
```

> **Les 3 clés xpack sont obligatoires.** Sans elles, la création de règles Indicator Match
> échoue silencieusement dans l'UI Kibana Security.

### Définir le mot de passe kibana_system

```bash
curl -k -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  -X POST "https://192.168.126.10:9200/_security/user/kibana_system/_password" \
  -H "Content-Type: application/json" \
  -d '{"password":"<MOT_DE_PASSE_KIBANA_SYSTEM>"}'
# Réponse attendue : {}
```

### Limiter la mémoire Kibana

```bash
echo "--max-old-space-size=512" | sudo tee -a /etc/kibana/node.options
```

### Démarrage

```bash
sudo systemctl enable kibana
sudo systemctl start kibana
# Démarrage : 2-3 minutes
```

Après démarrage : activer la **licence Trial** dans Kibana → Stack Management → License
Management → Start Trial (requis pour Kibana Security).

---

## 3. Logstash multi-pipeline

### Installation et tuning heap

```bash
sudo apt install logstash -y
```

Modifier `/etc/logstash/jvm.options` :

```
-Xms256m
-Xmx256m
```

### Architecture multi-pipeline

Le lab utilise deux pipelines séparés :

- **skoupa** : reçoit les events Beats sur le port 5044 et les envoie en interne
- **straight-es** : route chaque event vers le bon index Elasticsearch selon la source

Fichier de configuration : [`logstash/pipelines.yml`](logstash/pipelines.yml)  
Pipeline skoupa : [`logstash/skoupa.conf`](logstash/skoupa.conf)  
Pipeline straight-es : [`logstash/straight_es.conf`](logstash/straight_es.conf)

```bash
sudo mkdir -p /etc/logstash/pipeline
# Copier les 3 fichiers vers leurs emplacements système (voir entêtes des fichiers)
```

### Démarrage et validation

```bash
sudo systemctl enable logstash
sudo systemctl start logstash

# Valider que les 2 pipelines démarrent
sudo journalctl -fu logstash | grep "Pipeline started"
# Attendu :
#   Pipeline started {"pipeline.id"=>"straight-es"}
#   Pipeline started {"pipeline.id"=>"skoupa"}

# Valider le port 5044
ss -tlnp | grep 5044
```

---

## 4. Filebeat threatintel (sur VM ELK)

Filebeat est installé sur VM ELK exclusivement pour le module threatintel (IOCs MISP).
Il utilise `output.elasticsearch` direct — **pas Logstash** — pour que les ingest pipelines
ECS s'exécutent correctement et peuplent `threat.indicator.ip`.

```bash
sudo apt install filebeat -y
```

Configuration : [`filebeat-threatintel/filebeat.yml.example`](filebeat-threatintel/filebeat.yml.example)

```bash
# Charger les ingest pipelines dans Elasticsearch (obligatoire avant le premier démarrage)
sudo filebeat setup --pipelines --modules threatintel

sudo systemctl enable filebeat
sudo systemctl start filebeat
```

---

## 5. Data Views Kibana

Kibana → Stack Management → Data Views → Create data view :

| Nom | Index pattern | Timestamp |
|-----|---------------|-----------|
| SOC Logs | `soc-*` | @timestamp |
| Windows Logs | `soc-winlogbeat-*` | @timestamp |
| MISP IOCs | `filebeat-*` | event.ingested |

> Pour MISP IOCs, utiliser `event.ingested` comme timestamp pour éviter le problème
> des `@timestamp` MISP anciens (certains IOCs datent de 2014).

---

## 6. ILM Policy

Politique de rétention `soc-policy` :

- **Hot** : max 5 GB / max 3 jours
- **Warm** : 3 jours, readonly + shrink 1 shard + forcemerge 1 segment
- **Delete** : 14 jours

Fichier JSON : [`kibana/ilm_policy.json`](kibana/ilm_policy.json)

Appliquer via Kibana → Stack Management → Index Lifecycle Policies → Create policy.

Après création, appliquer aux index existants et créer un index template `soc-template`
avec pattern `soc-*` et `number_of_replicas: 0` (évite les warnings "yellow" en single-node).
