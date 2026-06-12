# Feeds MISP - Configuration et justifications

Les feeds que j'ai activés dans MISP → Sync Actions → Feeds, et pourquoi.

---

## Feeds activés

| Feed | Type | Format | Pourquoi je l'ai gardé |
|------|------|--------|------------------------|
| CIRCL OSINT Feed | MISP | MISP JSON | Qualité élevée, maintenu par les créateurs de MISP. Couvre IP, domaines, hashes, URLs avec contexte ATT&CK. |
| Feodo Tracker IP Blocklist | CSV | CSV | IOCs C2 botnet (Emotet, TrickBot, Dridex). Mis à jour quotidiennement. Peu de faux positifs. |
| abuse.ch URLhaus | MISP | MISP JSON | URLs malveillantes actives. Taux de mise à jour élevé, utile pour détecter les connexions sortantes. |
| Threatfox | MISP | MISP JSON | IOCs récents multi-types (IP, domaine, hash, URL). Complète CIRCL sur les menaces récentes. |
| MISP Warning Lists | - | - | Obligatoire. Exclut les faux positifs : CDNs, DNS publics (8.8.8.8), IPs cloud AWS/Azure/GCP. |

---

## Feeds que j'ai écartés

| Feed | Pourquoi |
|------|----------|
| AlienVault OTX | Volume trop élevé, qualité variable, beaucoup de faux positifs. Dépasse la capacité de stockage de ma VM (15 Go). |
| Bambenek C2 | Redondant avec Feodo pour les C2 que je couvre dans ce lab. |
| OpenPhish | URLs phishing uniquement - moins utile pour la détection réseau Sysmon Event ID 3. |
| Emerging Threats | Format snort/suricata, non compatible avec le module threatintel Filebeat. |
| Autres feeds MISP par défaut | Non vérifiés pour leur taille. Je n'active rien sans vérifier d'abord l'impact sur les 15 Go de disque. |

---

## Scheduled Tasks

Pour automatiser la mise à jour quotidienne :

MISP → Administration → Scheduled Tasks :

```
fetch_feeds  : Frequency 24h
cache_feeds  : Frequency 24h
```

---

## Volume approximatif après ingestion initiale

| Feed | IOCs type IP | Notes |
|------|-------------|-------|
| Feodo | ~500 | Très ciblé, haute fidélité |
| CIRCL OSINT | ~15 000+ | Multi-types, volume variable |
| URLhaus | ~5 000+ | Majoritairement URLs/domaines |
| Threatfox | ~3 000+ | Multi-types récents |
| **Total threat.indicator.ip** | **> 22 000** | Champ peuplé par les ingest pipelines Filebeat |

Le volume exact dépend de la fenêtre `var.first_interval` (720h = 30 jours). Pour vérifier :

```bash
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  "https://192.168.126.10:9200/filebeat-8.19.16/_count?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"exists":{"field":"threat.indicator.ip"}}}'
```
