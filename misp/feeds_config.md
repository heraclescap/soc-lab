# Feeds MISP — Configuration et justifications

Feeds activés dans MISP → Sync Actions → Feeds.

---

## Feeds activés

| Feed | Type | Format | Justification |
|------|------|--------|---------------|
| CIRCL OSINT Feed | MISP | MISP JSON | Qualité élevée, très complet, maintenu par les créateurs de MISP. Couvre IP, domaines, hashes, URLs avec contexte ATT&CK. |
| Feodo Tracker IP Blocklist | CSV | CSV | IOCs C2 botnet (Emotet, TrickBot, Dridex). Mis à jour quotidiennement. IPs très fiables, peu de faux positifs. |
| abuse.ch URLhaus | MISP | MISP JSON | URLs malveillantes actives (malware distribution, phishing). Taux de mise à jour élevé. Utile pour détecter les connexions sortantes. |
| Threatfox | MISP | MISP JSON | IOCs récents multi-types (IP, domaine, hash, URL). Communauté active. Complète CIRCL sur les menaces récentes. |
| MISP Warning Lists | — | — | **Obligatoire.** Exclut les faux positifs (CDNs, DNS publics, IPs cloud AWS/Azure/GCP). Sans elles, des alertes se déclenchent sur 8.8.8.8 et d'autres IPs légitimes. |

---

## Feeds écartés

| Feed | Raison de l'exclusion |
|------|----------------------|
| AlienVault OTX | Volume très élevé, qualité variable, beaucoup de faux positifs. Dépasse la capacité de stockage de la VM (15 Go). |
| Bambenek C2 | Redondant avec Feodo pour les C2 couverts dans ce lab. |
| OpenPhish | URLs phishing uniquement — moins pertinent pour la détection réseau Sysmon Event ID 3. |
| Emerging Threats | Format snort/suricata — non compatible natif avec le module threatintel Filebeat. |
| Autres feeds MISP par défaut | Non vérifiés pour leur taille. La VM a 15 Go de disque — ne pas activer sans vérification préalable. |

---

## Configuration Scheduled Tasks

Pour automatiser la mise à jour quotidienne des IOCs :

MISP → Administration → Scheduled Tasks :

```
fetch_feeds  : Frequency 24h
cache_feeds  : Frequency 24h
```

---

## Volumes approximatifs après ingestion initiale

| Feed | IOCs type IP | Notes |
|------|-------------|-------|
| Feodo | ~500 | Très ciblé, haute fidélité |
| CIRCL OSINT | ~15 000+ | Multi-types, volume variable |
| URLhaus | ~5 000+ | Majoritairement URLs/domaines |
| Threatfox | ~3 000+ | Multi-types récents |
| **Total threat.indicator.ip** | **> 22 000** | Champ peuplé par les ingest pipelines Filebeat |

> Le comptage exact dépend de la fenêtre temporelle de `var.first_interval` (720h = 30 jours).
> Pour vérifier : `curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' "https://192.168.126.10:9200/filebeat-8.19.16/_count?pretty" -H "Content-Type: application/json" -d '{"query":{"exists":{"field":"threat.indicator.ip"}}}'`
