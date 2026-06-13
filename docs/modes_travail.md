# Comment je travaille avec le lab

Je n'ai jamais fait tourner les 3 VMs en même temps - 16 Go de RAM ne suffisent pas. J'ai basculé entre deux modes selon les besoins.

---

## Mode config

Je l'ai utilisé pour mettre à jour les feeds MISP, créer ou affiner des règles de détection, analyser les IOCs.

```
VMs actives  : ELK (192.168.126.10) + MISP (192.168.126.20)
VM suspendue : Victim (192.168.126.30)
Filebeat ELK : actif - collecte les IOCs MISP toutes les 10 min
```

Ce que j'ai fait en mode config :
- Mettre à jour les feeds MISP (Fetch and store all events)
- Créer ou affiner des règles dans Kibana Security
- Vérifier les nouveaux IOCs dans la Data View `MISP IOCs` (filebeat-8.19.16)
- Analyser les alertes Indicator Match dans Kibana Security → Alerts
- Ajuster les règles prebuilt Elastic

---

## Mode attaque

Je l'ai utilisé pour les simulations Atomic Red Team, observer les alertes en temps réel, reconstruire des timelines.

```
VMs actives  : ELK (192.168.126.10) + Victim (192.168.126.30)
VM suspendue : MISP (192.168.126.20)
Filebeat ELK : arrêté - libère ~400 Mo de RAM
```

Ce que j'ai fait en mode attaque :
- Lancer une technique Atomic Red Team (PowerShell admin sur VM Victim)
- Observer les alertes Kibana Security en temps réel
- Reconstruire la timeline dans Kibana → Timeline
- Corréler Event IDs Sysmon → technique MITRE ATT&CK
- Écrire une règle EQL si aucune alerte n'est déclenchée (voir repo `soc-lab-detection-engineering`)
- Restaurer le snapshot "clean-sysmon-winlogbeat" avant la prochaine simulation

---

## Commandes de basculement

### Passer en mode attaque

Sur VM ELK :

```bash
sudo systemctl stop filebeat
free -m  # vérifier la RAM libérée
```

Puis dans VMware : suspendre MISP, démarrer/reprendre Victim.

### Revenir en mode config

Sur VM ELK :

```bash
sudo systemctl start filebeat
sudo journalctl -fu filebeat | grep "events published"  # vérifier que la collecte reprend
```

Puis dans VMware : suspendre Victim, reprendre MISP.

> VMware → VM → Suspend prend 2 secondes. La reprendre prend 5-10 secondes, contre 2-3 minutes pour un boot complet. J'ai toujours suspendu plutôt qu'éteint.

---

## Vérification santé du lab

À faire sur VM ELK après tout redémarrage ou changement de mode.

### État des services

```bash
sudo systemctl status elasticsearch kibana logstash | grep -E "Active|●"
```

Tous doivent être `active (running)`.

### Ordre de démarrage après reboot de la VM ELK

```bash
sudo systemctl start elasticsearch
sleep 30
sudo systemctl start kibana
sleep 20
sudo systemctl start logstash
# En mode config uniquement :
sudo systemctl start filebeat
```

### Pipelines Logstash actifs

```bash
sudo journalctl -u logstash | grep "Pipeline started" | tail -5
# Doit afficher skoupa et straight-es
```

### Espace disque

```bash
df -h /
# Je surveille si > 80% - les index soc-* grossissent pendant les simulations
```

### RAM disponible

```bash
free -m
# Mode config : ~1,5-2 Go libres sur 4 Go
# Mode attaque (sans Filebeat) : ~400 Mo de plus
```

### Index présents

```bash
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  "https://192.168.126.10:9200/_cat/indices/soc-*?v&s=index"
```

### IOCs MISP (mode config)

```bash
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  "https://192.168.126.10:9200/filebeat-8.19.16/_count?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"exists":{"field":"threat.indicator.ip"}}}'
# Attendu : count > 20000
```

### Alertes ouvertes

```bash
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  "https://192.168.126.10:9200/.alerts-security.alerts-default/_count?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"term":{"kibana.alert.workflow_status":"open"}}}'
```
