# Sysmon — Configuration et Event IDs

## Configuration utilisée

Config **SwiftOnSecurity** — référence communautaire pour les labs DFIR.

Source : https://github.com/SwiftOnSecurity/sysmon-config

```powershell
# Téléchargement et installation
$client = New-Object System.Net.WebClient
$client.DownloadFile(
    "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml",
    "C:\Sysmon\config.xml"
)
C:\Sysmon\Sysmon64.exe -accepteula -i C:\Sysmon\config.xml
```

---

## Event IDs capturés et usage DFIR

| Event ID | Nom | Techniques ATT&CK | Usage DFIR |
|----------|-----|-------------------|------------|
| 1 | Process Create | T1059, T1003 | Détection d'exécution de processus suspects, commandline |
| 3 | Network Connection | T1071, T1021 | Connexions C2, lateral movement — champ DestinationIp corrélé avec IOCs MISP |
| 7 | Image Loaded (DLL) | T1055 | DLL injection, process hollowing |
| 8 | CreateRemoteThread | T1055 | Injection de code dans un processus distant |
| 10 | ProcessAccess (LSASS) | T1003.001 | Credential dumping (Mimikatz, procdump sur lsass) |
| 11 | FileCreate | T1105 | Dépôt de fichiers (droppers, outils d'attaque) |
| 12/13 | Registry Events | T1547.001 | Persistence via Run keys, modifications registre |
| 15 | FileCreateStreamHash | T1105 | Alternate Data Streams, MOTW bypass |
| 22 | DNS Query | T1071.004 | DNS C2, tunneling DNS |

---

## Corrélation avec la règle Indicator Match Kibana

L'Event ID **3 (Network Connection)** est la source principale de l'alerte MISP IoC Match.

```
soc-winlogbeat-*
  winlog.event_data.DestinationIp
    MATCHES
  filebeat-8.19.16
    threat.indicator.ip
```

Sysmon Event ID 3 peuple `winlog.event_data.DestinationIp` et `winlog.event_data.SourceIp`
(via Winlogbeat → Logstash → soc-winlogbeat-*).

---

## Snapshot "clean"

Avant toute simulation Atomic Red Team, prendre un snapshot VMware :

```
Nom         : clean-sysmon-winlogbeat
Description : Windows 11 Pro - Sysmon + Winlogbeat + ART configurés - état propre
```

Restaurer après chaque simulation pour repartir d'un état propre.
Maintenir un seul snapshot actif pour économiser l'espace disque.

---

## PowerShell Script Block Logging

Activer via registre (PowerShell admin) :

```powershell
$regPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-Item -Path $regPath -Force
Set-ItemProperty -Path $regPath -Name "EnableScriptBlockLogging" -Value 1
```

Event ID 4104 capture le code PowerShell déobfusqué avant exécution.
Essentiel pour détecter les techniques Atomic Red Team via PowerShell encodé (`-EncodedCommand`).
