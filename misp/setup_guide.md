# MISP Setup Guide

Installation de MISP 2.4 sur VM Ubuntu 22.04 (192.168.126.20).

---

## Prérequis VM

```
OS    : Ubuntu 22.04 LTS Server
RAM   : 2048 Mo
CPU   : 2 cœurs
Disk  : 15 Go thin provisioned
Net   : VMnet8 NAT — IP statique 192.168.126.20/24
```

IP statique via `/etc/netplan/00-installer-config.yaml` :

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses: [192.168.126.20/24]
      routes:
        - to: default
          via: 192.168.126.2
      nameservers:
        addresses: [8.8.8.8]
```

---

## 1. Installation MISP

Le script officiel installe Apache, MariaDB, Redis, PHP et l'application MISP.
Durée estimée : **20-30 minutes**.

```bash
sudo apt update && sudo apt upgrade -y
wget -O /tmp/misp-install.sh https://raw.githubusercontent.com/MISP/MISP/2.4/INSTALL/INSTALL.sh
chmod +x /tmp/misp-install.sh
sudo /tmp/misp-install.sh -A
```

### Quirks du script d'installation — lire avant de lancer

**1. Mot de passe MariaDB régénéré à chaque run**

Le script régénère `database.php` avec un nouveau mot de passe aléatoire sans mettre à jour
l'utilisateur MariaDB. En cas d'échec et de re-run, l'utilisateur MariaDB est désynchronisé.

Workaround — recréer l'utilisateur manuellement :

```bash
sudo mysql -u root
```

```sql
DROP USER IF EXISTS 'misp'@'localhost';
CREATE USER 'misp'@'localhost' IDENTIFIED BY '<NOUVEAU_MOT_DE_PASSE>';
GRANT ALL PRIVILEGES ON misp.* TO 'misp'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**2. Boucle infinie sur `faup`**

Le clone git de la librairie `faup` peut boucler indéfiniment à ~68% de progression.
Interrompre avec **Ctrl+C** et continuer l'installation manuellement depuis l'étape suivante.

**3. Workers MISP manquants**

Après installation, vérifier que les 5 queues de workers tournent :
`default`, `prio`, `cache`, `email`, `update`.

La queue `default` est souvent absente au premier démarrage :

```bash
sudo systemctl restart misp-workers
sudo systemctl status misp-workers
```

---

## 2. Tuning MariaDB pour 2 Go de RAM

> Chemin de config MariaDB sur Ubuntu 22.04 : `/etc/mysql/mariadb.conf.d/50-server.cnf`
> (pas `/etc/mysql/mysql.conf.d/mysqld.cnf` qui est le chemin MySQL classique)

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Ajouter sous `[mysqld]` :

```ini
innodb_buffer_pool_size = 512M
innodb_log_file_size    = 64M
max_connections         = 50
```

> Ne pas définir `innodb_log_file_size` à une valeur inférieure aux fichiers de log
> existants — MariaDB refuserait de démarrer.

```bash
sudo systemctl restart mariadb
```

---

## 3. Premier accès

Depuis le navigateur Windows host : `https://192.168.126.20`

```
Login    : admin@admin.test
Password : admin
```

**Changer le mot de passe immédiatement** :
MISP → Administration → Edit profile → Change password

---

## 4. Feeds — voir `feeds_config.md`

Activer les feeds dans MISP → Sync Actions → Feeds.
Pour chaque feed activé, cliquer **Fetch and store all events**.

Scheduled Tasks pour mise à jour automatique :
MISP → Administration → Scheduled Tasks → `fetch_feeds` + `cache_feeds` : Frequency 24h

---

## 5. Clé API pour ELK

MISP → Administration → Auth Keys → Add authentication key

```
Permissions  : read-only
IP whitelist : 192.168.126.0/24
Commentaire  : ELK threatintel integration
```

> La whitelist doit couvrir la plage `/24` complète (pas seulement l'IP de la VM ELK).
> Mettre uniquement `192.168.126.10` fonctionne aussi mais la plage `/24` est plus robuste
> si des IPs changent.
>
> **Copier la clé générée immédiatement** — elle ne sera plus affichée après fermeture de la page.

La clé servira dans `elk/filebeat-threatintel/filebeat.yml.example` (`var.api_token`).

---

## 6. Validation de l'intégration MISP → ELK

Sur VM ELK, après démarrage du Filebeat threatintel :

```bash
# Compter les IOCs avec IP
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  "https://192.168.126.10:9200/filebeat-8.19.16/_count?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"exists":{"field":"threat.indicator.ip"}}}'
# Attendu : count > 20000 après quelques minutes d'ingestion
```
