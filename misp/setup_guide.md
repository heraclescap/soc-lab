# MISP Setup Guide

Installation de MISP 2.4 sur ma VM MISP - Ubuntu 22.04, IP 192.168.126.20.

---

## Prérequis VM

```
OS    : Ubuntu 22.04 LTS Server
RAM   : 2048 Mo
CPU   : 2 cœurs
Disk  : 15 Go thin provisioned
Net   : VMnet8 NAT - IP statique 192.168.126.20/24
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

Le script officiel installe Apache, MariaDB, Redis, PHP et l'application. Compter 20-30 minutes.

```bash
sudo apt update && sudo apt upgrade -y
wget -O /tmp/misp-install.sh https://raw.githubusercontent.com/MISP/MISP/2.4/INSTALL/INSTALL.sh
chmod +x /tmp/misp-install.sh
sudo /tmp/misp-install.sh -A
```

### Trois problèmes que j'ai rencontrés - à lire avant de lancer

**1. Mot de passe MariaDB régénéré à chaque run**

Le script régénère `database.php` avec un nouveau mot de passe aléatoire sans mettre à jour l'utilisateur MariaDB. Si le script échoue et que je le relance, MariaDB rejette toutes les connexions de MISP. Il faut recréer l'utilisateur manuellement :

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

Le clone git de `faup` peut boucler à ~68% sans jamais terminer. Ctrl+C et continuer manuellement.

**3. Workers MISP manquants**

Après installation, la queue `default` est souvent absente :

```bash
sudo systemctl restart misp-workers
sudo systemctl status misp-workers
```

Il doit y avoir 5 queues actives : `default`, `prio`, `cache`, `email`, `update`.

---

## 2. Tuning MariaDB pour 2 Go de RAM

Le chemin de config sur Ubuntu 22.04 est `/etc/mysql/mariadb.conf.d/50-server.cnf` et pas `/etc/mysql/mysql.conf.d/mysqld.cnf` (piège classique).

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Ajouter sous `[mysqld]` :

```ini
innodb_buffer_pool_size = 512M
innodb_log_file_size    = 64M
max_connections         = 50
```

> Ne pas descendre `innodb_log_file_size` en dessous de la taille des fichiers de log existants, MariaDB refuserait de démarrer.

```bash
sudo systemctl restart mariadb
```

---

## 3. Premier accès

Depuis mon navigateur Windows : `https://192.168.126.20`

```
Login    : admin@admin.test
Password : admin
```

Changer le mot de passe immédiatement : MISP → Administration → Edit profile → Change password.

---

## 4. Feeds - voir [`feeds_config.md`](feeds_config.md)

J'active les feeds dans MISP → Sync Actions → Feeds. Pour chaque feed activé, je clique **Fetch and store all events**.

Scheduled Tasks pour la mise à jour automatique :
MISP → Administration → Scheduled Tasks → `fetch_feeds` + `cache_feeds` en Frequency 24h.

---

## 5. Clé API pour ELK

MISP → Administration → Auth Keys → Add authentication key :

```
Permissions  : read-only
IP whitelist : 192.168.126.0/24
Commentaire  : ELK threatintel integration
```

J'utilise la plage `/24` entière plutôt que la seule IP de la VM ELK. Plus souple si les IPs évoluent.

La clé générée va dans `var.api_token` de [`../elk/filebeat-threatintel/filebeat.yml.example`](../elk/filebeat-threatintel/filebeat.yml.example). Elle ne s'affiche qu'une fois.

---

## 6. Validation

Sur VM ELK, après démarrage du Filebeat threatintel :

```bash
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  "https://192.168.126.10:9200/filebeat-8.19.16/_count?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"exists":{"field":"threat.indicator.ip"}}}'
# Attendu : count > 20000 après quelques minutes d'ingestion
```
