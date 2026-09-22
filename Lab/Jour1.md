# JOUR 1 - Enrichir la télémétrie Windows avec Sysmon

**Télémétrie :** c'est l'ensemble des données (métriques, états, bugs) collectées automatiquement sur un terminal et envoyées à distance pour surveiller son bon fonctionnement.

Pour donc enrechir les journaux d'évènemnents windows, je vais devoir installer Sysom<br>
Tout au long nous allons utiliser Powershell en mode administrateur dans windows

> Ressource wazuh `https://documentation.wazuh.com/current/user-manual/manager/event-logging.html`

Tout va commencer dans un premier directement au niveau de ma machine windows 11 client.

## Créer un dossier travail où tout seras centralisé niveau Windows 11 client

Ici, j'ai fais le choix de tout faire en ligne de commande avec powershell en mode administrateur, et le dossier de travail je l'ai appelé SOC-Lab

```powershell
New-Item -ItemType Directory -Path "C:\SOC-Lab" -Force
Get-Item "C:\SOC-Lab"
```

**Je vais donc télécharger Sysmon ici et le mettre dans SOC-Lab :**

`https://learn.microsoft.com/fr-fr/sysinternals/downloads/sysmon` ou `https://www.it-connect.fr/windows-utilisation-de-sysmon-pour-tracer-les-activites-malveillantes/`

**Je vais donc télécharger le fichier SwiftOnSecurity qui est un fichier très complet lié à Sysmon**

`https://github.com/SwiftOnSecurity/sysmon-config/blob/master/sysmonconfig-export.xml`

*ce fichier je le mettrai aussi dans C:/SOC-Lab/Sysmon*

**On vérifie donc la version Sysmon avec cette commande*

`.\Sysmon64.exe -s`

*On installe donc Sysmon*
`.\Sysmon64.exe -accepteula -i .\sysmonconfig-export.xml` et on vérifie que Sysmon fonctionne `Get-Service Sysmon64`

*On vérifie dans windows que le Journal sysmon réponds bien avec cette commande*

`Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 | Select-Object TimeCreated, Id, ProviderName`


