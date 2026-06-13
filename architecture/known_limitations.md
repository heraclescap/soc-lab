# Ce que j'ai appris à mes dépens

Tout ce qui m'a bloqué, surpris ou cassé pendant le déploiement. Je documente ça pour ne pas retomber dans les mêmes pièges, et parce que la plupart de ces comportements ne sont pas dans la doc officielle.

---

## Index `soc-*` au lieu de `logs-*`

Logstash ne peut pas écrire dans les data streams Elastic 8.x avec `op_type: index`. Mes premières tentatives avec `logs-*` produisaient des erreurs silencieuses - rien dans Kibana, rien de visible côté Logstash non plus.

J'ai choisi le préfixe `soc-` pour tous les index créés par Logstash. Et j'ai créé la Data View `soc-*` manuellement dans Kibana au lieu de compter sur les Data Views auto-générées par Elastic.

---

## Winlogbeat ne peuple pas `event.module`

Filebeat avec modules peuple `event.module` : `auditd`, `system`, etc. Winlogbeat ne le fait jamais. Mon routing Logstash basé sur `event.module` envoyait tous les events Windows dans `soc-unknown-*`.

La solution : vérifier `agent.type == "winlogbeat"` en premier dans le pipeline `straight_es.conf`, avant de regarder `event.module`. Voir les détails dans [elk/logstash/straight_es.conf](../elk/logstash/straight_es.conf).

---

## Filebeat threatintel exige un output Elasticsearch direct

Si Filebeat threatintel envoie vers Logstash, `filebeat setup --pipelines` échoue. Les ingest pipelines qui normalisent les IOCs MISP en champs ECS (`threat.indicator.ip`, etc.) ne s'exécutent que côté Elasticsearch. Résultat avec Logstash : `threat.indicator.ip` reste vide, la règle Indicator Match ne matche rien.

J'ai mis un Filebeat dédié sur la VM ELK avec `output.elasticsearch` direct, séparé du Filebeat de VM MISP qui lui utilise `output.logstash`.

---

## Data stream `filebeat-8.19.16` et pattern `filebeat-*`

En Elastic 8.x, Filebeat écrit dans un data stream dont le nom exact est `filebeat-8.19.16`. Le pattern `filebeat-*` fonctionne comme indicator index dans la règle Indicator Match - Kibana Security résout correctement le pattern vers le data stream sous-jacent.

Dans la règle Indicator Match, j'ai utilisé `filebeat-*` comme indicator index.

---

## Timestamps MISP anciens

Les IOCs MISP ont un `@timestamp` qui correspond à la date de création originale dans le feed source. Certains remontent à 2014. Dans Kibana Discover avec la fenêtre "Last 15 minutes", aucun IOC n'apparaît.

J'ai créé la Data View MISP IOCs avec `event.ingested` comme timestamp field plutôt que `@timestamp`. Sinon, l'option "All time" dans Kibana Discover fonctionne aussi.

---

## URL module threatintel : `/events/restSearch` uniquement

Le module Filebeat threatintel parse les attributs depuis la structure imbriquée `Event.Attribute`. L'endpoint `/attributes/restSearch` retourne une structure plate que le module ne sait pas découper - aucun IOC ne se crée.

Il faut obligatoirement `/events/restSearch`.

---

## Filtre temporel 30 jours sur les IOCs

Sans filtre temporel sur l'indicator index, les IOCs de 2016 génèrent des faux positifs sur des IPs réutilisées. J'ai mis `@timestamp >= "now-30d/d"` dans la règle. Les feeds Feodo et Threatfox mis à jour quotidiennement ont des timestamps récents et ne sont pas impactés.

---

## Règle Indicator Match : OR obligatoire entre DestinationIp et SourceIp

Avec AND entre les deux mappings, la règle exige que DestinationIp ET SourceIp matchent le même IOC en même temps. `SourceIp` est toujours 192.168.126.30 (la VM Victim). Résultat : zéro alerte.

Il faut OR entre les deux mappings.

---

## Auditd : `-p wa` uniquement

Mes règles auditd capturent les writes et attribute changes, pas les reads. `cat /etc/shadow` ne génère aucune alerte, c'est voulu. Avec `-p r` combiné à `-S execve`, le volume de logs explose - chaque lecture de fichier par chaque processus devient un événement.

---

## RAM ELK contrainte - 3,5 Go sur 4 Go alloués

Elasticsearch (1024 Mo heap) + Kibana (512 Mo) + Logstash (256 Mo) + Filebeat = environ 3,5 Go sur les 4 Go alloués. En mode attaque, j'arrête Filebeat avec `sudo systemctl stop filebeat` pour libérer ~400 Mo. Sans ça, la VM ELK commence à swapper.

---

## MISP script install - MariaDB désynchronisé après re-run

Le script officiel MISP régénère `database.php` avec un nouveau mot de passe aléatoire à chaque exécution, sans mettre à jour l'utilisateur MariaDB. Si le script échoue et que je le relance, MariaDB rejette toutes les connexions de MISP.

Il faut recréer l'utilisateur MariaDB manuellement après chaque re-run. Le chemin de config MariaDB sur Ubuntu 22.04 est `/etc/mysql/mariadb.conf.d/50-server.cnf`, pas `/etc/mysql/mysql.conf.d/mysqld.cnf`.

---

## Clés de chiffrement Kibana - échec silencieux

Sans les 3 clés `xpack.*.encryptionKey` dans `kibana.yml`, la création de règles Indicator Match échoue sans aucun message d'erreur dans l'UI. J'ai perdu du temps là-dessus.

Les 3 clés sont obligatoires, minimum 32 caractères chacune. Voir [elk/setup_guide.md](../elk/setup_guide.md).

---

## 1696 règles prebuilt actives toutes les 5 minutes

Kibana sature la RAM avec ce rythme sur une VM à 4 Go. J'ai réduit l'intervalle à 1h via bulk action dans Kibana Security. Aucun impact sur la détection dans un context lab.
