# Modes de travail du SOC Lab

Le lab fonctionne selon deux modes exclusifs — les 3 VMs ne tournent **jamais simultanément**
(contrainte RAM : 16 Go total sur le host Windows 11).

---

## Mode config

**Usage** : mise à jour des feeds MISP, création/affinage de règles de détection Kibana,
analyse des IOCs, consultation des alertes Indicator Match existantes.

```
VMs actives  : ELK (192.168.126.10) + MISP (192.168.126.20)
VM suspendue : Victim (192.168.126.30)
Filebeat ELK : actif — collecte des IOCs MISP toutes les 10 min
```

**Tâches typiques :**
- Mettre à jour les feeds MISP (Fetch and store all events)
- Créer ou affiner des règles de détection dans Kibana Security
- Vérifier les nouveaux IOCs dans la Data View `MISP IOCs` (filebeat-8.19.16)
- Analyser les alertes Indicator Match dans Kibana Security → Alerts
- Ajuster les règles prebuilt Elastic

---

## Mode attaque

**Usage** : simulation de techniques MITRE ATT&CK avec Atomic Red Team, observation des
alertes en temps réel, reconstruction de timelines, écriture de règles EQL.

```
VMs actives  : ELK (192.168.126.10) + Victim (192.168.126.30)
VM suspendue : MISP (192.168.126.20)
Filebeat ELK : arrêté — libère ~400 Mo de RAM nécessaires à la VM Victim
```

**Tâches typiques :**
- Lancer une technique Atomic Red Team (PowerShell admin sur VM Victim)
- Observer les alertes Kibana Security en temps réel
- Reconstruire la timeline dans Kibana → Timeline
- Corréler Event IDs Sysmon → technique MITRE ATT&CK
- Écrire une règle EQL si aucune alerte n'a été déclenchée (voir repo `soc-lab-detection`)
- Restaurer le snapshot "clean-sysmon-winlogbeat" avant la prochaine simulation

---

## Commandes de basculement

### Passer en mode attaque (sur VM ELK)

```bash
# 1. Arrêter Filebeat pour libérer ~400 Mo de RAM
sudo systemctl stop filebeat

# 2. Vérifier l'espace libéré
free -m
```

Puis dans VMware Workstation :
- Suspendre la VM MISP → VM → Suspend (2 secondes)
- Démarrer/Reprendre la VM Victim

### Revenir en mode config (sur VM ELK)

```bash
# 1. Redémarrer Filebeat pour reprendre la collecte IOCs
sudo systemctl start filebeat

# 2. Vérifier que Filebeat collecte bien
sudo journalctl -fu filebeat | grep "events published"
```

Puis dans VMware Workstation :
- Suspendre la VM Victim
- Reprendre la VM MISP (5-10 secondes vs 2-3 min pour un boot complet)

---

## Vérification santé du lab

À exécuter sur **VM ELK** après tout redémarrage ou changement de mode.

### État des services ELK

```bash
sudo systemctl status elasticsearch kibana logstash | grep -E "Active|●"
```

Tous doivent être `active (running)`.

### Ordre de démarrage (après reboot de la VM ELK)

```bash
sudo systemctl start elasticsearch
sleep 30
sudo systemctl start kibana
sleep 20
sudo systemctl start logstash
# En mode config uniquement :
sudo systemctl start filebeat
```

### Validation des pipelines Logstash

```bash
sudo journalctl -u logstash | grep "Pipeline started" | tail -5
# Doit afficher les 2 pipelines : skoupa et straight-es
```

### Espace disque

```bash
df -h /
# Surveiller si > 80% — les index soc-* grossissent pendant les simulations
```

### RAM disponible

```bash
free -m
# Mode config : ~1.5-2 Go libres sur 4 Go
# Mode attaque (sans Filebeat) : ~400 Mo de plus
```

### Index présents

```bash
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  "https://192.168.126.10:9200/_cat/indices/soc-*?v&s=index"
```

### Compter les IOCs MISP (mode config)

```bash
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  "https://192.168.126.10:9200/filebeat-8.19.16/_count?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"exists":{"field":"threat.indicator.ip"}}}'
# Attendu : count > 20000
```

### Alertes Kibana Security ouvertes

```bash
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  "https://192.168.126.10:9200/.alerts-security.alerts-default/_count?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"term":{"kibana.alert.workflow_status":"open"}}}'
```

---

## Utilisation VMware Suspend vs Shutdown

Toujours **suspendre** les VMs inutilisées plutôt que les éteindre :
- Suspend : 2 secondes → la VM reprend en 5-10 secondes
- Shutdown : RAM libérée, mais démarrage = 2-3 minutes + services à relancer

Exception : arrêt planifié long (> 8h) → shutdown pour économiser le disque.
