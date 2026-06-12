# Architecture réseau du SOC Lab

## Topologie VMnet8 NAT

Tout le lab est sur un unique segment VMnet8 NAT fourni par VMware Workstation.
Ce choix donne accès à Internet pour les feeds MISP et les téléchargements de paquets,
sans exposition publique des services (NAT masque les VMs derrière l'IP du host).

```
WINDOWS 11 HOST
│
└── VMnet8 (NAT) — 192.168.126.0/24
     Gateway   : 192.168.126.2   (VMware NAT engine)
     DNS (ext) : 8.8.8.8
     DHCP pool : .128 → .254     (réservé VMware — ne pas utiliser pour IPs statiques)
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

Toutes les IPs sont **statiques** (configurées dans `/etc/netplan/` sur Linux,
via `New-NetIPAddress` sur Windows). La plage DHCP VMware (.128-.254) n'est pas utilisée.

---

## Ports exposés et services

| Port | Protocole | VM | Service | Accessible depuis |
|------|-----------|----|---------|-------------------|
| 9200 | TCP | ELK | Elasticsearch REST API (TLS) | Host + VMs |
| 5601 | TCP | ELK | Kibana UI (HTTP) | Host + VMs |
| 5044 | TCP | ELK | Logstash Beats input | VM MISP + VM Victim |
| 443 | TCP | MISP | MISP Web UI (HTTPS) | Host |
| — | — | Victim | Aucun port exposé | — |

> Kibana est accessible depuis le navigateur Windows host à `http://192.168.126.10:5601`.  
> MISP est accessible à `https://192.168.126.20`.  
> Logstash n'accepte que les connexions Beats (port 5044) — sans TLS sur ce lab.

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

## Choix réseau justifiés

**Pourquoi VMnet8 NAT et pas VMnet1 Host-only**  
Les feeds MISP (Feodo, CIRCL, URLhaus, Threatfox) nécessitent un accès Internet direct
pour se mettre à jour. VMnet1 Host-only isolé ne permet pas cet accès. Le NAT maintient
l'isolement (les VMs ne sont pas accessibles de l'extérieur) tout en autorisant les
connexions sortantes.

**Pourquoi Logstash sans TLS sur le port 5044**  
Lab local NAT uniquement. Aucun trafic Beats ne traverse Internet. Activer TLS sur Logstash
ajoute une complexité de gestion des certificats sans bénéfice de sécurité dans ce contexte.

**Pourquoi IPs statiques et pas DHCP**  
Les configurations Filebeat/Winlogbeat référencent `192.168.126.10:5044` en dur.
Un changement d'IP DHCP casserait le pipeline de collecte. Les IPs statiques évitent
tout redémarrage de configuration après reboot des VMs.

