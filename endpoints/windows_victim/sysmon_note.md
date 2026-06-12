# Sysmon - Configuration et Event IDs

## Configuration utilisée

J'utilise la config **SwiftOnSecurity** - c'est la référence pour les labs DFIR, maintenue par la communauté.

Source : https://github.com/SwiftOnSecurity/sysmon-config

```powershell
$client = New-Object System.Net.WebClient
$client.DownloadFile(
    "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml",
    "C:\Sysmon\config.xml"
)
C:\Sysmon\Sysmon64.exe -accepteula -i C:\Sysmon\config.xml
```

---

## Event IDs capturés et usage DFIR

| Event ID | Nom | Techniques ATT&CK | Ce que je cherche |
|----------|-----|-------------------|-------------------|
| 1 | Process Create | T1059, T1003 | Exécution de processus suspects, arguments en ligne de commande |
| 3 | Network Connection | T1071, T1021 | Connexions C2, lateral movement - `DestinationIp` corrélé avec IOCs MISP |
| 7 | Image Loaded (DLL) | T1055 | DLL injection, process hollowing |
| 8 | CreateRemoteThread | T1055 | Injection de code dans un processus distant |
| 10 | ProcessAccess (LSASS) | T1003.001 | Credential dumping (Mimikatz, procdump sur lsass) |
| 11 | FileCreate | T1105 | Dépôt de fichiers : droppers, outils d'attaque |
| 12/13 | Registry Events | T1547.001 | Persistence via Run keys |
| 15 | FileCreateStreamHash | T1105 | Alternate Data Streams, MOTW bypass |
| 22 | DNS Query | T1071.004 | DNS C2, tunneling DNS |

---

## Corrélation avec la règle Indicator Match Kibana

L'Event ID 3 (Network Connection) est la source principale de mes alertes MISP IoC Match. Sysmon peuple `winlog.event_data.DestinationIp` et `winlog.event_data.SourceIp`, qui sont ensuite corrélés avec `threat.indicator.ip` dans `filebeat-8.19.16`.

```
soc-winlogbeat-*
  winlog.event_data.DestinationIp  OR  winlog.event_data.SourceIp
    MATCHES
  filebeat-8.19.16
    threat.indicator.ip
```

---

## Snapshot "clean"

Avant toute simulation Atomic Red Team, je prends un snapshot VMware :

```
Nom         : clean-sysmon-winlogbeat
Description : Windows 11 Pro - Sysmon + Winlogbeat + ART configurés - état propre
```

Je restore après chaque simulation. Je maintiens un seul snapshot actif pour économiser l'espace disque.

---

## PowerShell Script Block Logging

```powershell
$regPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-Item -Path $regPath -Force
Set-ItemProperty -Path $regPath -Name "EnableScriptBlockLogging" -Value 1
```

L'Event ID 4104 capture le code PowerShell déobfusqué avant exécution. C'est indispensable pour les techniques Atomic Red Team qui passent par PowerShell encodé (`-EncodedCommand`).
