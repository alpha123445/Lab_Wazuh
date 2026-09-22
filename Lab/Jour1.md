# JOUR 1 - Enrichir la télémétrie Windows avec Sysmon

**Télémétrie :** c'est l'ensemble des données (métriques, états, bugs) collectées automatiquement sur un terminal et envoyées à distance pour surveiller son bon fonctionnement.

Pour donc enrechir les journaux d'évènemnents windows, je vais devoir installer Sysom

> Ressource wazuh `https://documentation.wazuh.com/current/user-manual/manager/event-logging.html`

Tout va commencer dans un premier directement au niveau de ma machine windows 11 client.

## Créer un dossier travail où tu seras centralisé niveau Windows 11 client

Ici, j'ai fais le choix de tout faire en ligne de commande avec powershell en mode administrateur, et le dossier de travail je l'ai appelé SOC-Lab

```powershell
New-Item -ItemType Directory -Path "C:\SOC-Lab" -Force
Get-Item "C:\SOC-Lab"
```
