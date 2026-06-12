# Limitations et learnings documentés

Résumé honnête des compromis et problèmes rencontrés lors du déploiement du lab.
Ces limitations sont **intentionnellement conservées** pour documenter les décisions
d'architecture et aider à déboguer des comportements inattendus.

---

## Index `soc-*` au lieu de `logs-*`

**Problème** : Logstash ne peut pas écrire dans les data streams Elastic 8.x avec
`op_type: index`. Tentatives d'écriture dans `logs-*` produisent des erreurs d'indexation
silencieuses.

**Décision** : préfixe `soc-` pour tous les index créés par Logstash (`soc-winlogbeat-*`,
`soc-auditd-*`, `soc-system-*`). Utiliser la Data View `soc-*` créée manuellement dans
Kibana au lieu des Data Views auto-générées par Elastic.

---

## Winlogbeat ne peuple pas `event.module`

**Problème** : Contrairement à Filebeat avec modules (qui peuple `event.module: "auditd"`,
`event.module: "system"`, etc.), Winlogbeat ne peuple jamais le champ `event.module`.
Le routing Logstash basé sur `event.module` laisse tous les events Winlogbeat tomber dans
`soc-unknown-*`.

**Décision** : routing Logstash basé sur `agent.type == "winlogbeat"` en priorité.
Voir `elk/logstash/straight_es.conf` pour l'implémentation.

---

## Filebeat threatintel nécessite un output Elasticsearch direct

**Problème** : `filebeat setup --pipelines` échoue si l'output configuré est Logstash.
Les ingest pipelines Elasticsearch (qui transforment les IOCs MISP en champs ECS
`threat.indicator.ip`) s'exécutent uniquement côté Elasticsearch — ils ne s'exécutent
jamais si les events passent par Logstash. Résultat : `threat.indicator.ip` reste vide,
la règle Indicator Match ne matche rien.

**Décision** : Filebeat threatintel sur VM ELK avec `output.elasticsearch` direct.
Un Filebeat dédié (séparé du Filebeat de VM MISP qui lui utilise `output.logstash`).

---

## Data stream `filebeat-8.19.16` vs pattern `filebeat-*`

**Problème** : En Elastic 8.x, Filebeat écrit dans un data stream dont le nom exact est
`filebeat-8.19.16`. Le pattern `filebeat-*` dans les règles Kibana Security **ne matche pas**
les backing indices `.ds-filebeat-8.19.16-YYYY.MM.dd-000001`.

**Décision** : utiliser le nom exact `filebeat-8.19.16` comme indicator index dans la règle
Indicator Match (pas `filebeat-*`).

---

## Timestamps MISP anciens

**Problème** : Les IOCs MISP ont un `@timestamp` dérivé de la date de création originale
dans le feed source — parfois 2014, 2016, 2018. Dans Kibana Discover avec la fenêtre
temporelle par défaut "Last 15 minutes", aucun IOC n'apparaît.

**Décision** : créer la Data View MISP IOCs avec `event.ingested` comme timestamp field
(date d'ingestion dans Elasticsearch). Ou utiliser le filtre "All time" dans Kibana Discover.

---

## URL module threatintel : `/events/restSearch` et non `/attributes/restSearch`

**Problème** : Le module Filebeat threatintel parse les attributs IOC depuis la structure
imbriquée `Event.Attribute`. L'endpoint `/attributes/restSearch` retourne une structure
plate incompatible avec le split du module — aucun IOC n'est créé.

**Décision** : utiliser `/events/restSearch` comme configuré dans
`elk/filebeat-threatintel/filebeat.yml.example`.

---

## Filtre temporel IOCs : 30 jours (`now-30d/d`)

**Constat** : Les IOCs anciens (> 30 jours) ont une probabilité très faible d'être encore
actifs. Sans filtre temporel, les IOCs de 2016 génèrent des faux positifs sur des IPs
réutilisées depuis.

**Décision** : filtre `@timestamp >= "now-30d/d"` sur l'indicator index dans la règle
Indicator Match. Les feeds Feodo et Threatfox mis à jour quotidiennement ont des `@timestamp`
récents et ne sont pas impactés.

---

## Règle Indicator Match : OR obligatoire entre DestinationIp et SourceIp

**Problème** : avec AND entre les deux mappings, la règle exige que les deux champs
(`DestinationIp` ET `SourceIp`) matchent le même IOC simultanément. `SourceIp` est toujours
l'IP de la VM Victim (192.168.126.30) — jamais un IOC. Résultat : zéro alerte.

**Décision** : mapping OR entre `winlog.event_data.DestinationIp` et
`winlog.event_data.SourceIp` dans la règle. Un seul des deux champs suffit pour déclencher
une alerte.

---

## Auditd : `-p wa` uniquement — lectures non capturées

**Constat** : Les règles auditd du lab utilisent `-p wa` (write + attribute change).
`cat /etc/shadow` ou `sudo cat /etc/passwd` ne génèrent **pas** d'alerte.

**Décision intentionnelle** : `-p r` combiné avec `-S execve` génère un volume de logs
très élevé (toute lecture de fichier = événement). Acceptable pour un lab orienté
détection d'écriture, de persistence et d'exécution.

---

## RAM ELK contrainte — 3.5 Go sur 4 Go alloués

**Constat** :
- Elasticsearch heap : 1024 Mo (réduit de 1500 Mo pour coexister avec les autres services)
- Kibana heap : 512 Mo
- Logstash heap : 256-512 Mo
- Filebeat : ~100-400 Mo
- Total : ~3.5 Go sur 4 Go alloués

**Contrainte** : ne jamais faire tourner les 3 VMs simultanément (ELK + MISP + Victim).
En mode attaque, arrêter Filebeat (`sudo systemctl stop filebeat`) pour libérer ~400 Mo.

---

## MISP script install — MariaDB régénéré à chaque run

**Problème** : le script officiel MISP régénère `database.php` avec un nouveau mot de passe
aléatoire à chaque exécution, sans mettre à jour l'utilisateur MariaDB. En cas d'échec et
de re-run, MariaDB rejette les connexions de MISP.

**Workaround** : recréer manuellement l'utilisateur MariaDB après un re-run du script.
Chemin de config MariaDB sur Ubuntu 22.04 : `/etc/mysql/mariadb.conf.d/50-server.cnf`
(pas `/etc/mysql/mysql.conf.d/mysqld.cnf`).

---

## Clés de chiffrement Kibana — échec silencieux

**Problème** : sans les 3 clés `xpack.*.encryptionKey` dans `kibana.yml`, la création de
règles Indicator Match dans Kibana Security **échoue silencieusement** — aucun message
d'erreur visible dans l'UI.

**Décision** : les 3 clés sont obligatoires (minimum 32 caractères chacune).
Voir `elk/setup_guide.md` pour la procédure de génération.

---

## Règles prebuilt Elastic — saturation RAM

**Constat** : 1696 règles actives toutes les 5 minutes saturent la RAM de Kibana sur
une VM à 4 Go.

**Décision** : réduire l'intervalle de toutes les règles prebuilt à 1h via bulk action.
Facteur de réduction : 12x. Aucun impact sur la détection dans un contexte lab.

