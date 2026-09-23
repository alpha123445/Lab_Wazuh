# JOUR 1 -  Enrichir la télémétrie Windows avec Sysmon

## Objectif du jour

Pour commencer ce projet, je vais d'abord travailler sur ma machine **Windows 11 Client**.

L'objectif de cette première journée est d'enrichir les journaux Windows afin d'avoir davantage d'informations exploitables par la suite dans Wazuh. Pour cela, je vais installer **Sysmon**, le configurer avec une configuration de référence, puis connecter ses journaux à l'agent Wazuh déjà installé sur ma machine.

La logique que je cherche à mettre en place est donc la suivante :

```text
Action réalisée sur Windows
        ↓
       Sysmon
        ↓
Événement Windows / Event ID
        ↓
   Wazuh Agent
        ↓
   Wazuh Manager
        ↓
Wazuh Dashboard
        ↓
  Analyse SOC
```

### Qu'est-ce que la télémétrie ?

La télémétrie correspond à l'ensemble des données collectées automatiquement sur un système afin de pouvoir suivre son activité et son état.

Dans le contexte d'un SOC, cela peut notamment être :

- des créations de processus ;
- des connexions réseau ;
- des créations ou modifications de fichiers ;
- des modifications du registre ;
- des événements d'authentification ;
- des requêtes DNS ;
- etc.

Plus la télémétrie est pertinente, plus il est possible de reconstruire ce qui s'est réellement passé sur une machine.

### Ressource utilisée

Documentation Wazuh :

`https://documentation.wazuh.com/current/user-manual/manager/event-logging.html`

---

# 1. Préparer la machine Windows 11

Tout au long de cette partie, je vais utiliser **PowerShell en mode administrateur**.

Je fais ici le choix de réaliser les principales étapes en ligne de commande afin de pouvoir reproduire plus facilement le lab par la suite.

## 1.1 Créer un dossier de travail

Je commence par créer un dossier qui servira à centraliser les fichiers liés au projet sur Windows.

J'ai choisi de l'appeler :

```text
C:\SOC-Lab
```

Dans PowerShell administrateur :

```powershell
New-Item -ItemType Directory -Path "C:\SOC-Lab" -Force
```

Je vérifie ensuite que le dossier a bien été créé :

```powershell
Get-Item "C:\SOC-Lab"
```

Je vais utiliser ce dossier tout au long du projet pour conserver les outils, configurations et fichiers de test.

---

# 2. Télécharger et installer Sysmon

## 2.1 Télécharger Sysmon

Je vais utiliser **Sysmon**, fourni par Microsoft Sysinternals.

Ressource officielle :

`https://learn.microsoft.com/fr-fr/sysinternals/downloads/sysmon`

Une autre ressource utile pour comprendre son fonctionnement :

`https://www.it-connect.fr/windows-utilisation-de-sysmon-pour-tracer-les-activites-malveillantes/`

Je télécharge donc l'archive Sysmon, puis je l'extrais dans :

```text
C:\SOC-Lab\Sysmon
```

Je dois obtenir une organisation ressemblant à ceci :

```text
C:\SOC-Lab\
└── Sysmon\
    ├── Sysmon64.exe
    └── ...
```

Pour mon Windows 11, j'utiliserai normalement :

```text
Sysmon64.exe
```

---

# 3. Utiliser une configuration de référence Sysmon

Sysmon peut fonctionner avec une configuration personnalisée. Plutôt que de partir de zéro, j'ai choisi d'utiliser la configuration de référence proposée par **SwiftOnSecurity**.

Cette configuration permet d'avoir une base déjà travaillée pour observer différents comportements système.

Ressource :

`https://github.com/SwiftOnSecurity/sysmon-config/blob/master/sysmonconfig-export.xml`

Je récupère le fichier :

```text
sysmonconfig-export.xml
```

et je le place ici :

```text
C:\SOC-Lab\Sysmon\sysmonconfig-export.xml
```

J'obtiens donc :

```text
C:\SOC-Lab\
└── Sysmon\
    ├── Sysmon64.exe
    └── sysmonconfig-export.xml
```

L'idée est d'utiliser cette configuration comme **base de travail**, puis de l'adapter plus tard en fonction des besoins du lab.

---

# 4. Vérifier Sysmon avant l'installation

Je me rends dans le dossier Sysmon :

```powershell
cd "C:\SOC-Lab\Sysmon"
```

Je peux ensuite afficher le schéma de configuration supporté par ma version de Sysmon :

```powershell
.\Sysmon64.exe -s
```

Cela me permet notamment de vérifier que l'exécutable fonctionne correctement avant d'aller plus loin.

---

# 5. Installer Sysmon avec notre configuration

Je lance maintenant l'installation :

```powershell
.\Sysmon64.exe -accepteula -i .\sysmonconfig-export.xml
```

Sysmon va alors installer son service et appliquer la configuration fournie.

Une fois terminé, je vérifie que le service fonctionne :

```powershell
Get-Service Sysmon64
```

Je dois obtenir un résultat indiquant :

```text
Status  : Running
Name    : Sysmon64
```

À ce stade, Sysmon est installé sur mon endpoint Windows 11.

---

# 6. Vérifier que Windows reçoit bien les événements Sysmon

Avant de connecter Sysmon à Wazuh, je veux déjà vérifier que **Sysmon fonctionne directement au niveau de Windows**.

Pour cela, je consulte le journal :

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 |
    Select-Object TimeCreated, Id, ProviderName
```

Je devrais voir apparaître des événements provenant de :

```text
Microsoft-Windows-Sysmon
```

avec différents Event IDs.

Les journaux peuvent également être consultés graphiquement avec :

> **Observateur d'événements Windows**

Puis :

```text
Journaux des applications et des services
└── Microsoft
    └── Windows
        └── Sysmon
            └── Operational
```

C'est ici que Sysmon écrit ses événements.

---

# 7. Générer nous-mêmes quelques événements

Maintenant que Sysmon fonctionne, je vais générer quelques actions simples afin de vérifier qu'elles sont bien enregistrées.

L'objectif n'est pas encore de réaliser une attaque. Je veux simplement comprendre la chaîne :

```text
ACTION
   ↓
WINDOWS
   ↓
SYSMON
   ↓
EVENT ID
```

## 7.1 Test - Création d'un processus

Je lance le Bloc-notes :

```powershell
notepad.exe
```

Puis je peux vérifier les événements de création de processus avec :

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 30 |
    Where-Object Id -eq 1 |
    Select-Object -First 5 TimeCreated, Id, Message
```

Je cherche notamment l'exécution de :

```text
notepad.exe
```

L'**Event ID 1** correspond ici à la création d'un processus.

---

## 7.2 Test - Exécution de PowerShell

Je vais maintenant générer un autre événement en lançant PowerShell :

```powershell
powershell.exe -NoProfile -Command "Write-Host 'SOC LAB TEST'"
```

Puis je vérifie à nouveau les événements de création de processus :

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 50 |
    Where-Object Id -eq 1 |
    Select-Object -First 10 TimeCreated, Id
```

Je dois pouvoir retrouver le lancement de PowerShell dans les événements.

---

## 7.3 Test - Création d'un fichier

Je vais maintenant créer un fichier dans mon dossier de travail :

```powershell
New-Item -Path "C:\SOC-Lab\test.txt" -ItemType File -Force
```

Puis je recherche les événements correspondant à la création de fichiers :

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 50 |
    Where-Object Id -eq 11 |
    Select-Object -First 10 TimeCreated, Id, Message
```

Je cherche notamment un événement contenant le fichier :

```text
C:\SOC-Lab\test.txt
```

L'**Event ID 11** correspond ici à la création d'un fichier.

---

# 8. La logique que je cherche à comprendre

Je fais volontairement ces petits tests avant de connecter Wazuh, car je veux d'abord comprendre ce que chaque action produit comme trace.

Pour l'instant :

```text
ACTION
  ↓
WINDOWS
  ↓
SYSMON
  ↓
EVENT ID
```

Par exemple :

```text
notepad.exe
   ↓
Sysmon
   ↓
Event ID 1
```

ou :

```text
Création de test.txt
   ↓
Sysmon
   ↓
Event ID 11
```

La prochaine étape consiste à faire remonter cette télémétrie vers mon SIEM.

Je veux donc arriver à :

```text
ACTION
  ↓
WINDOWS
  ↓
SYSMON
  ↓
WAZUH AGENT
  ↓
WAZUH MANAGER
  ↓
WAZUH DASHBOARD
  ↓
SOC ANALYST
```

---

# 9. Relier maintenant Sysmon à Wazuh

J'ai déjà un **agent Wazuh fonctionnel sur mon Windows 11**.

Je vais donc maintenant modifier la configuration de cet agent afin qu'il collecte le journal Sysmon.

## 9.1 Localiser le fichier `ossec.conf`

Sur Windows, le fichier de configuration de l'agent Wazuh se trouve normalement ici :

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Avant de le modifier, je vais faire une sauvegarde.

---

# 10. Sauvegarder `ossec.conf`

Dans PowerShell administrateur :

```powershell
Copy-Item `
"C:\Program Files (x86)\ossec-agent\ossec.conf" `
"C:\Program Files (x86)\ossec-agent\ossec.conf.bak"
```

Je vérifie que la sauvegarde existe :

```powershell
Get-Item "C:\Program Files (x86)\ossec-agent\ossec.conf*"
```

Je dois retrouver notamment :

```text
ossec.conf
ossec.conf.bak
```

Cette sauvegarde me permettra de revenir facilement à la configuration précédente en cas de problème.

---

# 11. Modifier `ossec.conf`

J'ouvre maintenant le fichier de configuration.

Avec Visual Studio Code :

```powershell
code "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

Je vais dans la balise :

```xml
<ossec_config>
```

et j'ajoute le bloc suivant à l'intérieur :

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Ce bloc indique à l'agent Wazuh qu'il doit surveiller le Windows Event Channel :

```text
Microsoft-Windows-Sysmon/Operational
```

avec le format :

```text
eventchannel
```

Je sauvegarde ensuite le fichier.

---

# 12. Redémarrer l'agent Wazuh

Après avoir modifié `ossec.conf`, je redémarre le service Wazuh :

```powershell
Restart-Service -Name Wazuh
```

Puis je vérifie qu'il fonctionne toujours :

```powershell
Get-Service Wazuh
```

Je dois retrouver :

```text
Status  : Running
```

Si le service ne redémarre pas, je ne vais pas continuer : je dois d'abord vérifier la configuration XML de `ossec.conf`.

---

# 13. Générer de nouveaux événements après l'intégration

Je vais maintenant refaire plusieurs actions afin de vérifier que les nouveaux événements sont également récupérés par Wazuh.

### Création d'un processus

```powershell
notepad.exe
```

### Création d'un fichier

```powershell
New-Item -Path "C:\SOC-Lab\wazuh-sysmon-test.txt" -ItemType File -Force
```

### Exécution de PowerShell

```powershell
powershell.exe -NoProfile -Command "Write-Host 'WAZUH-SYSMON-TEST'"
```

L'objectif est maintenant de vérifier que ces événements ne sont plus seulement visibles localement dans Windows, mais qu'ils arrivent également dans Wazuh.

---

# 14. Vérification dans Wazuh

Dernière étape de cette première partie : je me rends dans le **Wazuh Dashboard**.

Je recherche mon agent Windows et je vérifie les événements liés à Sysmon.

<img width="1887" height="1012" alt="image" src="https://github.com/user-attachments/assets/2826a3da-0d66-49e8-bbc4-04e6035b6987" />

<br>

<img width="1882" height="1012" alt="image" src="https://github.com/user-attachments/assets/616e4a01-b375-4a4e-acac-92c7753c4936" />



## Conclusion

Pour cette première journée, j'ai commencé par enrichir la télémétrie de mon endpoint Windows 11 avec Sysmon.

J'ai installé Sysmon avec une configuration de référence SwiftOnSecurity, puis j'ai vérifié directement dans Windows que les événements étaient bien générés et que je pouvais les identifier à travers leurs Event IDs.

J'ai ensuite configuré l'agent Wazuh afin qu'il récupère le journal `Microsoft-Windows-Sysmon/Operational` et transmette ces événements au Wazuh Manager.

Je dispose maintenant d'une base de télémétrie beaucoup plus intéressante pour la suite du projet. La prochaine étape sera de mieux maîtriser cette télémétrie, notamment en regardant plus précisément les événements qui seront utiles pour notre scénario d'attaque et notre travail de détection SOC.
