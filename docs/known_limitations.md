# Limitations du lab — guide d'usage quotidien

Ce que vous devez savoir pour travailler efficacement avec le lab.
Chaque point décrit un comportement potentiellement surprenant et comment le gérer.

---

## "Je ne vois aucun IOC dans Kibana Discover"

**Cause probable** : la fenêtre temporelle de Kibana est "Last 15 minutes" (par défaut)
et les `@timestamp` des IOCs MISP sont la date de création originale dans le feed source
(parfois 2014-2022, pas la date d'ingestion).

**Solution** :
1. Dans Kibana Discover avec la Data View `MISP IOCs`, passer à "All time"
2. Ou éditer la Data View pour utiliser `event.ingested` comme timestamp field

---

## "L'index soc-winlogbeat ne reçoit plus d'events Windows"

**Vérification rapide** :

```bash
# Sur VM ELK — voir si Logstash reçoit des events Winlogbeat
sudo journalctl -u logstash | grep "winlogbeat" | tail -10

# Vérifier que Logstash écoute sur 5044
ss -tlnp | grep 5044
```

Sur VM Victim (PowerShell) :
```powershell
Get-Service winlogbeat
# Doit être Running. Sinon : Start-Service winlogbeat
```

**Rappel** : si Winlogbeat envoie vers Logstash et que les events arrivent dans
`soc-unknown-*` plutôt que `soc-winlogbeat-*`, vérifier le pipeline `straight_es.conf` —
le routing via `agent.type == "winlogbeat"` doit être en première condition.

---

## "La règle Indicator Match ne génère aucune alerte"

Points à vérifier dans l'ordre :

1. **Index indicator** : l'indicator index doit être `filebeat-8.19.16` (pas `filebeat-*`)
2. **Mapping OR** : les deux mappings (DestinationIp et SourceIp) doivent être reliés par OR
3. **Filtre temporel IOCs** : `@timestamp >= "now-30d/d"` — si les IOCs ont des timestamps
   très anciens, ce filtre les exclut. Vérifier avec `event.ingested >= "now-30d/d"` à la place.
4. **threat.indicator.ip vide** : vérifier que le Filebeat threatintel utilise bien
   `output.elasticsearch` direct (pas `output.logstash`) et que les ingest pipelines ont été
   chargés avec `sudo filebeat setup --pipelines --modules threatintel`
5. **Règle désactivée** : Kibana Security → Rules → vérifier que la règle est "Enabled"

---

## "La création de règle dans Kibana Security échoue silencieusement"

**Cause** : les 3 clés `xpack.*.encryptionKey` sont absentes de `/etc/kibana/kibana.yml`.

**Solution** : générer 3 clés et les ajouter :

```bash
openssl rand -hex 32  # répéter 3 fois
sudo nano /etc/kibana/kibana.yml
# Ajouter :
# xpack.encryptedSavedObjects.encryptionKey: "<CLE_32_CHARS>"
# xpack.security.encryptionKey: "<CLE_32_CHARS>"
# xpack.reporting.encryptionKey: "<CLE_32_CHARS>"
sudo systemctl restart kibana
```

---

## "Kibana est très lent / OOM"

**Cause probable en mode attaque** : Filebeat threatintel tourne encore et consomme ~400 Mo.

```bash
sudo systemctl stop filebeat
# Libère ~400 Mo immédiatement
```

**Autre cause** : 1696 règles prebuilt actives toutes les 5 minutes.
Réduire à 1h via Kibana Security → Rules → Elastic rules → Select all → Bulk actions
→ Update rule schedules → 1h.

---

## "Les index soc-* n'existent pas dans Kibana"

**Cause probable** : aucun event n'a encore été reçu par Logstash (index créé au premier event).

**Solution** :
1. Vérifier que Winlogbeat ou Filebeat MISP envoie bien vers Logstash :5044
2. Générer un événement : `sudo touch /etc/shadow` sur VM MISP, ou attendre les logs
   Windows natifs sur VM Victim
3. Si les index existent mais pas dans Kibana : recréer la Data View `soc-*`

---

## "auditd ne génère pas d'alerte sur cat /etc/shadow"

**Comportement attendu** — les règles auditd du lab utilisent `-p wa` uniquement.
Les lectures ne sont pas capturées. `cat /etc/shadow` = lecture → aucune alerte.

Pour capturer les lectures : ajouter `-p r` aux règles concernées dans
`/etc/audit/rules.d/dfir.rules`. Attention au volume : `-S execve -p r` génère un
événement pour chaque lecture de fichier par chaque processus.

---

## "MISP ne répond plus après un restart"

**Vérifier les workers** — la queue `default` est souvent la coupable :

```bash
sudo systemctl status misp-workers
sudo systemctl restart misp-workers
```

**Vérifier MariaDB** :

```bash
sudo systemctl status mariadb
# Si arrêté : sudo systemctl start mariadb
```

---

## "Les index sont trop grands / espace disque insuffisant"

Les index `soc-*` sont gérés par la policy ILM `soc-policy` :
- Hot : max 5 GB ou 3 jours
- Warm : passage à 3 jours
- Delete : suppression à 14 jours

Pour forcer la rotation des index en phase ILM :

```bash
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  -X POST "https://192.168.126.10:9200/soc-*/_ilm/retry"
```

Pour voir l'état ILM des index :

```bash
curl -sk -u elastic:'<MOT_DE_PASSE_ELASTIC>' \
  "https://192.168.126.10:9200/soc-*/_ilm/explain?pretty"
```

---

## "Winlogbeat écrit dans soc-unknown au lieu de soc-winlogbeat"

**Cause** : le pipeline `straight_es.conf` ne détecte pas `agent.type == "winlogbeat"`.

Vérifier la configuration en cours sur Logstash :

```bash
sudo cat /etc/logstash/pipeline/straight_es.conf
# La première condition doit être : if [agent][type] == "winlogbeat"
sudo systemctl restart logstash
```

---

## Rappels sur les contraintes RAM

| Mode | VMs actives | RAM consommée | Marge |
|------|------------|---------------|-------|
| Config | ELK + MISP | ~10 Go | 6 Go |
| Attaque | ELK + Victim | ~12 Go | 4 Go |
| **3 VMs** | **ELK + MISP + Victim** | **~14 Go** | **2 Go — RISQUÉ** |

Ne jamais faire tourner les 3 VMs simultanément.
