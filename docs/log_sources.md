# Sources de logs du SOC Lab

---

## Vue d'ensemble des sources

| Source | VM | Agent | Index ES | Protocole |
|--------|----|-------|----------|-----------|
| Sysmon (Event ID 1,3,7,8,10,11,12,13,15,22) | Victim | Winlogbeat | `soc-winlogbeat-*` | Logstash :5044 |
| Security Windows | Victim | Winlogbeat | `soc-winlogbeat-*` | Logstash :5044 |
| PowerShell Script Block | Victim | Winlogbeat | `soc-winlogbeat-*` | Logstash :5044 |
| WMI Activity | Victim | Winlogbeat | `soc-winlogbeat-*` | Logstash :5044 |
| Task Scheduler | Victim | Winlogbeat | `soc-winlogbeat-*` | Logstash :5044 |
| Windows Defender | Victim | Winlogbeat | `soc-winlogbeat-*` | Logstash :5044 |
| System / Application | Victim | Winlogbeat | `soc-winlogbeat-*` | Logstash :5044 |
| Auditd (syscalls) | MISP | Filebeat | `soc-auditd-*` | Logstash :5044 |
| Syslog / auth.log | MISP | Filebeat | `soc-system-*` | Logstash :5044 |
| IOCs MISP (threatintel) | ELK | Filebeat | `filebeat-8.19.16` | ES direct |

---

## Event IDs Windows capturés

### Sysmon (Microsoft-Windows-Sysmon/Operational)

| Event ID | Description | Technique ATT&CK |
|----------|-------------|------------------|
| 1 | Process Create | T1059, T1003 |
| 3 | Network Connection | T1071, T1021 |
| 7 | Image Loaded (DLL) | T1055 |
| 8 | CreateRemoteThread | T1055 |
| 10 | ProcessAccess (LSASS) | T1003.001 |
| 11 | FileCreate | T1105 |
| 12 | RegistryEvent (Object create/delete) | T1547.001 |
| 13 | RegistryEvent (Value Set) | T1547.001 |
| 15 | FileCreateStreamHash | T1105 |
| 22 | DNS Query | T1071.004 |

### Security Windows

| Event ID | Description |
|----------|-------------|
| 4624 | Connexion réussie |
| 4625 | Échec de connexion |
| 4648 | Connexion avec credentials explicites (runas) |
| 4688 | Création de processus |
| 4698 | Tâche planifiée créée |
| 4720 | Compte utilisateur créé |
| 4732 | Membre ajouté à un groupe local |
| 4756 | Membre ajouté à un groupe universel |
| 4768 | Ticket Kerberos (TGT) demandé |
| 4769 | Ticket Kerberos (service) demandé |

### PowerShell

| Event ID | Source | Description |
|----------|--------|-------------|
| 4103 | Microsoft-Windows-PowerShell/Operational | Module logging |
| 4104 | Microsoft-Windows-PowerShell/Operational | Script Block Logging (code déobfusqué) |
| 4105 | Microsoft-Windows-PowerShell/Operational | Command start |
| 4106 | Microsoft-Windows-PowerShell/Operational | Command stop |
| 400 | Windows PowerShell | Engine lifecycle |
| 403 | Windows PowerShell | Engine stopped |
| 600 | Windows PowerShell | Provider start |
| 800 | Windows PowerShell | Pipeline execution |

---

## Règles auditd actives (VM MISP)

| Règle | Clé | Type d'événement capturé |
|-------|-----|--------------------------|
| `-w /etc/passwd -p wa` | identity_modification | Modification du fichier passwd |
| `-w /etc/shadow -p wa` | identity_modification | Modification du fichier shadow |
| `-w /etc/sudoers -p wa` | privilege_escalation | Modification des sudoers |
| `-w /etc/crontab -p wa` | persistence | Modification de la crontab |
| `-a always,exit -F arch=b64 -S execve` | exec_commands | Toute exécution de commande |
| `-a always,exit -F arch=b64 -S connect` | network_connections | Connexions réseau sortantes |
| `-w /tmp -p x` | suspicious_exec | Exécution depuis /tmp |

**Note** : `-p wa` uniquement (write + attribute change). Les lectures ne génèrent pas d'alerte.

---

## Volume IOCs MISP

| Feed | Type | Volume approximatif |
|------|------|---------------------|
| CIRCL OSINT Feed | MISP JSON | 15 000+ IOCs |
| Feodo Tracker | CSV (IPs C2) | ~500 IPs |
| URLhaus | MISP JSON | 5 000+ URLs |
| Threatfox | MISP JSON | 3 000+ IOCs |
| **Total threat.indicator.ip** | — | **> 22 000** |

Les IOCs sont dans l'index `filebeat-8.19.16` (data stream Elasticsearch).
Le champ `threat.indicator.ip` est peuplé par les ingest pipelines Filebeat threatintel.

---

## Index templates et ILM

| Template | Pattern | ILM Policy |
|----------|---------|------------|
| `soc-template` | `soc-*` | `soc-policy` |

Policy `soc-policy` : hot 5 GB/3j → warm 3j (readonly+shrink+forcemerge) → delete 14j.
