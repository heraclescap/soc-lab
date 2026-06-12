# Architecture réseau

## Topologie VMnet8 NAT

J'ai mis tout le lab sur un seul segment VMnet8 NAT. C'est le choix le plus simple qui fonctionne : accès Internet pour les feeds MISP, et les VMs restent invisibles depuis l'extérieur.

```
WINDOWS 11 HOST
│
└── VMnet8 (NAT) - 192.168.126.0/24
     Gateway   : 192.168.126.2   (VMware NAT engine)
     DNS (ext) : 8.8.8.8
     DHCP pool : .128 → .254     (réservé VMware - ne pas utiliser pour IPs statiques)
     │
     ├── VM ELK      192.168.126.10
     ├── VM MISP     192.168.126.20
     └── VM Victim   192.168.126.30
```

---

## Adressage IP des VMs

| VM | IP | OS | RAM | Disque |
|----|----|----|-----|--------|
| ELK | 192.168.126.10 | Ubuntu 22.04 LTS Server | 4 Go | 25 Go thin |
| MISP | 192.168.126.20 | Ubuntu 22.04 LTS Server | 2 Go | 15 Go thin |
| Victim | 192.168.126.30 | Windows 11 Pro | 4 Go | 40 Go thin |

Toutes les IPs sont statiques. J'ai configuré ça via `/etc/netplan/` sur Linux et `New-NetIPAddress` sur Windows. Si je laissais le DHCP, les configs Filebeat/Winlogbeat qui pointent sur `192.168.126.10:5044` en dur cesseraient de fonctionner au prochain reboot.

---

## Ports exposés et services

| Port | Protocole | VM | Service | Accessible depuis |
|------|-----------|----|---------|-------------------|
| 9200 | TCP | ELK | Elasticsearch REST API (TLS) | Host + VMs |
| 5601 | TCP | ELK | Kibana UI (HTTP) | Host + VMs |
| 5044 | TCP | ELK | Logstash Beats input | VM MISP + VM Victim |
| 443 | TCP | MISP | MISP Web UI (HTTPS) | Host |
| - | - | Victim | Aucun port exposé | - |

Depuis mon navigateur Windows : Kibana sur `http://192.168.126.10:5601`, MISP sur `https://192.168.126.20`.

---

## Flux réseau entre VMs

```
VM MISP (192.168.126.20)
  → TCP 5044 → VM ELK  (Filebeat → Logstash)

VM Victim (192.168.126.30)
  → TCP 5044 → VM ELK  (Winlogbeat → Logstash)

VM ELK (192.168.126.10)
  → TCP 443  → VM MISP  (Filebeat threatintel → MISP /events/restSearch)
  → TCP 9200 → localhost (Logstash → Elasticsearch)

Host Windows
  → TCP 5601 → VM ELK  (Kibana UI)
  → TCP 443  → VM MISP (MISP UI)
```

---

## Pourquoi ces choix

**VMnet8 NAT et pas VMnet1 Host-only** : les feeds MISP (Feodo, CIRCL, URLhaus, Threatfox) ont besoin d'Internet pour se mettre à jour. VMnet1 Host-only coupe ça. Le NAT garde les VMs isolées tout en autorisant les connexions sortantes.

**Logstash sans TLS sur le port 5044** : c'est un lab local NAT, aucun trafic Beats ne passe sur Internet. Ajouter TLS sur Logstash demanderait de gérer des certificats pour un gain de sécurité nul dans ce contexte.

**IPs statiques** : les configs Filebeat et Winlogbeat référencent `192.168.126.10:5044` en dur. Un changement DHCP suffirait à casser l'ingestion de logs.
