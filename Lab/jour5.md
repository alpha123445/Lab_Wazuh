# JOUR 5 - Investigation SOC et reconstitution de la chaîne d'attaque

## Objectif du jour

Lors des quatre premiers jours, j'ai préparé la télémétrie de mon environnement puis simulé une attaque contrôlée.

- Le **Jour 1**, j'ai installé Sysmon sur Windows 11 et configuré l'agent Wazuh pour collecter le journal `Microsoft-Windows-Sysmon/Operational`.
- Le **Jour 2**, j'ai enrichi la télémétrie Debian avec auditd, SSH, Apache et le File Integrity Monitoring (FIM).
- Le **Jour 3**, j'ai simulé l'exécution d'un script PowerShell, une persistance par Registry Run Key et la récupération d'identifiants fictifs stockés dans un fichier.
- Le **Jour 4**, j'ai poursuivi le scénario avec une connexion SSH vers Debian, une opération de maintenance autorisée par sudo et la création d'un marqueur PHP inoffensif dans le répertoire Web Apache.

Aujourd'hui, je vais prendre le rôle de l'analyste SOC qui doit comprendre ce qui s'est passé à partir des événements disponibles.

**L'objectif n'est pas encore de créer les règles de détection personnalisées.** Je vais d'abord retrouver les traces, examiner leurs champs, corréler les sources et documenter les éventuelles lacunes de télémétrie. La création des règles personnalisées sera traitée au Jour 6.

La chaîne que je cherche à reconstituer est la suivante :

```text
KALI
  |
  | Hébergement du script de test
  v
WINDOWS 11
  |
  | Téléchargement et exécution PowerShell
  | Création de fichiers
  | Modification d'une Registry Run Key
  | Récupération d'identifiants fictifs
  v
CONNEXION SSH
  |
  v
DEBIAN
  |
  | Authentification du compte labops
  | Opération de maintenance avec euid=0
  | Création de soc-lab-status.php
  v
APACHE + AUDITD + FIM
  |
  v
WAZUH
  |
  v
TIMELINE D'INCIDENT
```

Les identifiants de ce scénario sont fictifs et la page PHP est un marqueur inoffensif. Le rapport devra préciser ces limites : il ne s'agit ni d'un vol réel de secrets Windows ni du déploiement d'un véritable webshell.

---

# 1. Préparer l'investigation

## 1.1 Identifier les machines et les adresses IP

Avant de rechercher des événements, je note les informations de mon environnement. Je ne dois pas utiliser les exemples ci-dessous sans les remplacer par les valeurs réelles de mon lab.

Sur Windows, dans PowerShell :

```powershell
hostname
ipconfig
```

Sur Debian :

```bash
hostname
hostname -I
```

Sur Kali :

```bash
hostname
hostname -I
```

Je remplis ce tableau dans mes notes :

| Élément | Valeur réelle de mon lab |
|---|---|
| Nom de l'agent Windows | `[À compléter]` |
| IP de Windows | `[À compléter]` |
| Nom de l'agent Debian | `[À compléter]` |
| IP de Debian | `[À compléter]` |
| IP de Kali | `[À compléter]` |
| Date et heure de la simulation | `[À compléter]` |
| Fuseau horaire utilisé | `[À compléter]` |

Je relève également les noms d'agents qui apparaissent dans Wazuh. Les noms utilisés dans les exemples suivants (`windows-agent` et `debian-agent`) sont des valeurs fictives : je dois les remplacer par les noms affichés dans mon Dashboard.

## 1.2 Définir la période à examiner

Dans Wazuh Dashboard, je règle la plage de temps pour qu'elle couvre les jours où j'ai réellement exécuté les tests. Si les actions datent de plusieurs jours, je choisis une plage personnalisée plutôt que de conserver uniquement « Last 24 hours ».

Je note l'heure de début et l'heure de fin de l'investigation. Cela évite de conclure trop rapidement qu'un événement est absent alors qu'il se trouve simplement en dehors de la période sélectionnée.

## 1.3 Comprendre la différence entre événements et alertes

Wazuh collecte des événements, puis les analyse avec des décodeurs et des règles. Un événement ne produit une alerte que si une règle correspond et que le niveau atteint le seuil de génération d'alertes.

Par défaut, `wazuh-alerts-*` contient les événements ayant déclenché une alerte. Les événements qui ne déclenchent aucune alerte ne sont pas nécessairement visibles dans cet index.

Les archives Wazuh peuvent conserver tous les événements reçus, y compris ceux qui ne déclenchent pas d'alerte, mais l'archivage et l'indexation des archives doivent être configurés. La documentation officielle explique cette différence :

- https://documentation.wazuh.com/current/user-manual/manager/event-logging.html
- https://documentation.wazuh.com/current/user-manual/wazuh-indexer/wazuh-indexer-indices.html

Je commence donc par les alertes déjà disponibles et les journaux locaux des endpoints. Je n'active pas l'archivage sans vérifier la configuration existante et l'espace disque du manager.

---

# 2. Vérifier les traces locales avant de chercher dans Wazuh

Cette étape sert de point de comparaison. Si un événement existe sur l'endpoint mais n'apparaît pas dans le Dashboard, je pourrai ensuite investiguer la collecte ou l'indexation.

## 2.1 Windows — Vérifier les fichiers et la Run Key

Sur Windows 11, dans PowerShell, je vérifie les principaux artefacts créés au Jour 3 :

```powershell
$LabPaths = @(
    "$env:TEMP\invoice_update.ps1",
    "C:\ProgramData\SOC-Lab\Stage\execution-marker.txt",
    "C:\ProgramData\SOC-Lab\Stage\persistence.ps1",
    "C:\ProgramData\SOC-Lab\Stage\stolen-credentials.txt"
)

foreach ($Path in $LabPaths) {
    [PSCustomObject]@{
        Path   = $Path
        Exists = Test-Path $Path
    }
}
```

Je vérifie la valeur de persistance :

```powershell
Get-ItemProperty `
  "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" |
  Select-Object OneDriveUpdateLab
```

Je note les fichiers absents et les résultats, mais je ne recrée pas les artefacts avant d'avoir terminé cette première vérification. Je veux analyser ce qui existe réellement.

## 2.2 Windows — Vérifier les événements Sysmon localement

### Event ID 1 — Création de processus

```powershell
Get-WinEvent `
  -FilterHashtable @{
      LogName = "Microsoft-Windows-Sysmon/Operational"
      Id      = 1
  } -MaxEvents 200 |
  Where-Object { $_.Message -match 'invoice_update\.ps1|powershell\.exe' } |
  Select-Object TimeCreated, Id, Message
```

### Event ID 11 — Création de fichier

```powershell
Get-WinEvent `
  -FilterHashtable @{
      LogName = "Microsoft-Windows-Sysmon/Operational"
      Id      = 11
  } -MaxEvents 200 |
  Where-Object { $_.Message -match 'SOC-Lab|invoice_update\.ps1|persistence\.ps1|stolen-credentials\.txt' } |
  Select-Object TimeCreated, Id, Message
```

### Event ID 13 — Modification d'une valeur du registre

```powershell
Get-WinEvent `
  -FilterHashtable @{
      LogName = "Microsoft-Windows-Sysmon/Operational"
      Id      = 13
  } -MaxEvents 200 |
  Where-Object { $_.Message -match 'OneDriveUpdateLab|CurrentVersion\\Run' } |
  Select-Object TimeCreated, Id, Message
```

Ces commandes lisent les événements conservés localement dans le journal Sysmon. Si aucune sortie n'apparaît, je vérifie d'abord si l'action a bien été exécutée, si le journal contient encore les événements et si la configuration Sysmon enregistre le type d'événement recherché.

## 2.3 Debian — Vérifier les journaux du Jour 4

Sur Debian, je vérifie la connexion SSH :

```bash
sudo grep -E 'sshd.*(labops|Accepted|Failed)' /var/log/auth.log | tail -n 40
```

Si `/var/log/auth.log` n'existe pas sur mon installation, j'utilise le journal systemd :

```bash
sudo journalctl -u ssh --since "24 hours ago"
```

Je recherche ensuite les événements auditd portant la clé définie au Jour 2 :

```bash
sudo ausearch -k soc_lab_exec -ts today -i
```

Si je veux cibler une période différente, j'adapte le filtre temporel d'`ausearch` à la date réelle du test.

Je vérifie le journal de maintenance créé au Jour 4 :

```bash
sudo tail -n 20 /var/log/soc-lab-maintenance.log
```

Puis le fichier PHP de test :

```bash
sudo stat /var/www/html/soc-lab-status.php
sudo cat /var/www/html/soc-lab-status.php
```

Enfin, je recherche la requête HTTP dans Apache :

```bash
sudo grep 'soc-lab-status.php' /var/log/apache2/access.log | tail -n 20
```

Ces vérifications me donnent une référence locale avant l'investigation dans le SIEM.

---

# 3. Ouvrir Wazuh et rechercher les alertes disponibles

Je me connecte au Wazuh Dashboard et j'ouvre la vue d'investigation des événements ou des alertes (le libellé exact peut dépendre de la version installée).

Je sélectionne l'index d'alertes `wazuh-alerts-*` et la plage de dates correspondant à ma simulation.

Je commence par filtrer sur le nom réel de mon agent Windows. Ensuite, je fais la même chose pour Debian. Je ne mets pas tout de suite plusieurs critères très précis dans la requête : je veux d'abord confirmer que des données existent pour les agents et la période choisis.

Exemple de filtre, à adapter avec le véritable nom de l'agent :

```text
agent.name: "windows-agent"
```

Puis :

```text
agent.name: "debian-agent"
```

Si le filtre sur l'agent renvoie des résultats, j'ajoute les critères techniques présentés dans les sections suivantes.

**À retenir :** l'absence d'une alerte ne démontre pas que l'événement n'a pas été collecté. Je vérifie aussi les journaux locaux et, si l'archivage est configuré, les archives.

---

# 4. Investiguer Windows dans Wazuh

## 4.1 Rechercher les créations de processus — Sysmon Event ID 1

Dans la barre de recherche Wazuh, je peux commencer par :

```text
agent.name: "windows-agent" AND data.win.system.eventID: 1
```

Je remplace `windows-agent` par le nom réel de mon agent.

Dans les résultats, j'ouvre un événement et j'examine les informations disponibles, notamment :

- `Image` : chemin du programme lancé ;
- `CommandLine` : ligne de commande ;
- `ParentImage` : programme parent ;
- `User` : compte associé à l'exécution ;
- `ProcessGuid` et `ParentProcessGuid` : identifiants permettant de distinguer les processus dans la télémétrie Sysmon.

Les noms de champs visibles dans le JSON peuvent être représentés sous la forme `data.win.eventdata.image`, `data.win.eventdata.commandLine`, etc. Je me fie au JSON de l'événement reçu plutôt qu'à un nom de champ supposé.

Je cherche notamment une trace cohérente avec l'exécution de `invoice_update.ps1` ou de PowerShell au moment du test.

## 4.2 Affiner la recherche sur le script du Jour 3

Si les champs correspondants existent dans mon index, j'essaie :

```text
agent.name: "windows-agent" AND data.win.system.eventID: 1 AND data.win.eventdata.commandLine: *invoice_update.ps1*
```

Si cette requête ne donne aucun résultat, je retire le critère sur `commandLine` et j'inspecte les événements Event ID 1 manuellement. Les champs peuvent varier selon les événements, la configuration et les données réellement indexées.

Je relève l'heure, le nom du processus, la ligne de commande et le compte associé. Je ne considère pas une exécution PowerShell comme malveillante à elle seule : c'est son contexte et sa relation avec les autres événements qui rendent la séquence intéressante.

## 4.3 Rechercher la création des fichiers — Sysmon Event ID 11

Je recherche les événements de création de fichier :

```text
agent.name: "windows-agent" AND data.win.system.eventID: 11
```

Dans les résultats, je cherche les chemins liés au dossier :

```text
C:\ProgramData\SOC-Lab\Stage\
```

et les noms suivants :

```text
execution-marker.txt
persistence.ps1
stolen-credentials.txt
```

Le champ utile est généralement `TargetFilename` dans le JSON de l'événement Sysmon. Je relève le chemin exact et l'heure de création.

## 4.4 Rechercher la modification du registre — Sysmon Event ID 13

Je recherche les événements de modification de valeurs de registre :

```text
agent.name: "windows-agent" AND data.win.system.eventID: 13
```

J'ouvre les événements autour de la période du Jour 3 et je cherche :

```text
OneDriveUpdateLab
```

ainsi que :

```text
CurrentVersion\Run
```

Je relève le chemin de registre concerné, la valeur écrite, le processus à l'origine de la modification et le compte utilisé, lorsque ces champs sont disponibles.

La chaîne à vérifier est :

```text
Création du processus PowerShell
        ↓
Création de fichiers de test
        ↓
Modification de la Registry Run Key
        ↓
Récupération des identifiants fictifs
```

## 4.5 Examiner l'Event ID 10 sans prétendre avoir réalisé un credential dumping

L'Event ID 10 de Sysmon correspond à l'accès à un processus. Il peut être utile à une investigation portant sur un accès inhabituel à la mémoire d'un autre processus, mais il ne prouve pas à lui seul une tentative de vol d'identifiants.

Je peux le consulter avec :

```text
agent.name: "windows-agent" AND data.win.system.eventID: 10
```

Je regarde notamment les processus source et cible et les champs d'accès disponibles. Dans ce projet, le Credential Access réalisé au Jour 3 est une simulation par fichier ; je ne dois donc pas écrire dans le rapport que j'ai détecté ou réalisé un dump LSASS si cela n'a pas été fait.

---

# 5. Investiguer la connexion SSH vers Debian

## 5.1 Rechercher les alertes SSH

Dans Wazuh, je sélectionne l'agent Debian et je cherche les alertes associées à SSH. Selon les événements et les règles ayant déclenché une alerte, je peux commencer avec :

```text
agent.name: "debian-agent" AND rule.groups: sshd
```

Je remplace le nom de l'agent et j'inspecte le JSON d'un événement trouvé afin d'identifier les champs réellement disponibles pour l'utilisateur, l'adresse IP source et le résultat de l'authentification.

Si aucun résultat n'apparaît dans `wazuh-alerts-*`, je ne conclus pas immédiatement que SSH n'a pas été collecté. Je compare avec les journaux locaux de Debian et je garde à l'esprit qu'une authentification réussie n'est pas forcément une alerte de sécurité.

## 5.2 Vérifier la preuve locale

Sur Debian :

```bash
sudo grep -E 'sshd.*(labops|Accepted|Failed)' /var/log/auth.log | tail -n 40
```

Ou :

```bash
sudo journalctl -u ssh --since "24 hours ago"
```

Je recherche une ligne d'authentification réussie pour `labops` et je relève son horodatage ainsi que l'adresse IP source.

Je peux ensuite faire le lien avec la connexion sortante lancée depuis Windows, si l'événement Sysmon correspondant a été collecté.

La preuve à rechercher est :

```text
Machine source : Windows 11
Compte distant : labops
Machine cible  : Debian
Protocole      : SSH
Résultat       : Authentification réussie
```

Je ne remplis ces champs qu'avec des valeurs réellement observées dans les journaux.

---

# 6. Investiguer l'opération privilégiée avec auditd

## 6.1 Examiner les événements auditd sur Debian

Au Jour 2, j'ai configuré une clé auditd appelée `soc_lab_exec` pour surveiller certaines exécutions de programmes.

Je lance :

```bash
sudo ausearch -k soc_lab_exec -ts today -i
```

Je recherche l'exécution de :

```text
/usr/local/sbin/soc-lab-maintenance
```

Je m'intéresse particulièrement aux champs suivants lorsqu'ils sont présents :

| Champ | Utilité pour l'investigation |
|---|---|
| `auid` | Identité de connexion à l'origine de l'action, quand elle est renseignée |
| `uid` | Identité utilisateur associée à l'événement |
| `euid` | Identité effective utilisée pour les permissions |
| `exe` | Chemin de l'exécutable |
| `comm` | Nom de commande |
| `key` | Clé de la règle auditd ayant généré l'événement |

Je cherche à déterminer si l'action a bien été initiée par `labops` et exécutée avec les privilèges root. Les champs exacts dépendent du type d'événement auditd et de la règle qui l'a généré.

## 6.2 Vérifier le résultat de l'opération

Je vérifie le journal du script :

```bash
sudo tail -n 20 /var/log/soc-lab-maintenance.log
```

Je cherche la ligne contenant les informations :

```text
action=soc-lab-maintenance
caller=labops
euid=0
```

Cette ligne est un marqueur explicite écrit par notre script de laboratoire. Elle complète les événements auditd, mais ne les remplace pas : dans une investigation réelle, je privilégierais les traces système indépendantes et je vérifierais leur cohérence.

---

# 7. Investiguer la création du fichier dans le Webroot Apache

## 7.1 Vérifier la trace FIM dans Wazuh

Lors du Jour 2, j'ai configuré le File Integrity Monitoring pour surveiller `/var/www/html`.

Dans Wazuh, je recherche les alertes FIM de mon agent Debian :

```text
agent.name: "debian-agent" AND rule.groups: syscheck
```

Dans les résultats, je cherche le chemin :

```text
/var/www/html/soc-lab-status.php
```

Le champ contenant le chemin peut être `syscheck.path`, selon la structure de l'événement. J'ouvre le JSON pour vérifier le chemin réel, le type de changement (création, modification ou suppression) et les informations d'intégrité disponibles.

Si rien ne s'affiche, je vérifie d'abord que le fichier a été créé après l'activation du FIM, que `/var/www/html` est bien surveillé et que l'agent est actif. Une absence d'alerte peut aussi venir d'une différence entre les événements alertés et les événements simplement collectés.

## 7.2 Vérifier les informations de fichier sur Debian

Sur Debian :

```bash
sudo stat /var/www/html/soc-lab-status.php
```

Puis :

```bash
sudo sha256sum /var/www/html/soc-lab-status.php
```

Je note le propriétaire, les permissions et l'empreinte SHA-256. Le hash permet d'identifier précisément le fichier au moment de la capture ; il ne suffit pas, à lui seul, à prouver qu'un fichier est malveillant.

Documentation Wazuh sur le FIM :

- https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/how-it-works.html
- https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/how-to-configure-fim.html

---

# 8. Investiguer l'accès HTTP dans Apache

## 8.1 Rechercher la requête dans le journal local

Sur Debian :

```bash
sudo grep 'soc-lab-status.php' /var/log/apache2/access.log | tail -n 20
```

Je relève :

- l'horodatage ;
- l'adresse IP source ;
- la méthode HTTP ;
- le chemin demandé ;
- le code de réponse HTTP.

Si le test du Jour 4 a bien été réalisé depuis Kali, l'adresse IP source de la requête devrait correspondre à celle de Kali. Je vérifie cela avec les valeurs de mon environnement, sans supposer que c'est nécessairement le cas.

## 8.2 Vérifier si l'événement est visible dans Wazuh

Dans le Dashboard, je commence par sélectionner l'agent Debian et la période correcte.

Les requêtes HTTP qui répondent normalement ne génèrent pas nécessairement une alerte. Si `soc-lab-status.php` n'apparaît pas dans `wazuh-alerts-*`, je compare d'abord avec `access.log` sur Debian. Pour rechercher les événements bruts dans le Dashboard, l'index d'archives doit être configuré et alimenté.

Je ne confonds donc pas ces deux constats :

- **Apache a enregistré la requête localement** ;
- **Wazuh a généré une alerte pour cette requête**.

Ce sont deux choses différentes.

---

# 9. Si j'ai besoin de rechercher les événements bruts dans Wazuh

Cette section est à utiliser uniquement si les journaux locaux montrent bien les événements, mais qu'ils ne sont pas consultables dans le Dashboard parce qu'ils n'ont déclenché aucune alerte.

Les archives Wazuh peuvent conserver tous les événements reçus au format JSON. Cependant, il faut vérifier leur configuration avant de la modifier, car l'indexation de tous les événements augmente la consommation de stockage.

## 9.1 Vérifier l'état actuel côté manager Ubuntu

Sur le manager Wazuh :

```bash
sudo grep -n -E 'logall|logall_json' /var/ossec/etc/ossec.conf
```

Je vérifie aussi si un fichier d'archives JSON existe :

```bash
sudo ls -lh /var/ossec/logs/archives/archives.json
```

Si le fichier existe, je peux y rechercher un événement récent sans modifier la configuration :

```bash
sudo grep -F 'soc-lab-status.php' /var/ossec/logs/archives/archives.json | tail -n 10
```

Cette recherche dépend de la présence effective des archives et de la valeur recherchée dans le JSON. Si rien ne ressort, cela ne prouve pas à lui seul que l'événement n'a jamais été reçu.

## 9.2 Activer les archives si cela est nécessaire au lab

Avant d'activer l'archivage, je vérifie l'espace disque :

```bash
df -h
```

Je sauvegarde la configuration du manager :

```bash
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak
```

Dans le bloc `<global>` déjà présent dans `/var/ossec/etc/ossec.conf`, je règle ou ajoute une seule fois :

```xml
<logall_json>yes</logall_json>
```

Je ne crée pas un deuxième bloc `<global>` s'il en existe déjà un. Je redémarre ensuite le manager :

```bash
sudo systemctl restart wazuh-manager
sudo systemctl is-active wazuh-manager
```

L'activation de `logall_json` permet au manager d'écrire les événements en JSON dans `archives.json`. Pour les rechercher dans le Dashboard, il faut également activer l'indexation des archives dans la configuration Filebeat, si elle n'est pas déjà activée. Cette partie dépend de la configuration existante de l'installation, je la vérifie donc avant de la modifier.

Documentation officielle :

https://documentation.wazuh.com/current/user-manual/wazuh-indexer/wazuh-indexer-indices.html

**Attention :** je n'active pas l'archivage complet inutilement dans un environnement de production. Cela augmente le volume de logs et les besoins de stockage.

---

# 10. Construire la timeline de l'incident

Après avoir examiné les événements Windows et Debian, je remplis une chronologie à partir des horodatages réellement observés. Je ne fabrique pas d'heures pour combler les cases vides.

| Heure observée | Machine | Source | Événement | Interprétation | Preuve |
|---|---|---|---|---|---|
| `[À compléter]` | Kali | Serveur HTTP | Téléchargement demandé par Windows, si journalisé | Mise à disposition du script | Log du serveur Python / capture |
| `[À compléter]` | Windows | Sysmon ID 1 | Exécution de PowerShell | Exécution du script de test | Événement Sysmon |
| `[À compléter]` | Windows | Sysmon ID 11 | Création du ou des fichiers du lab | Artefacts de la simulation | Événement Sysmon |
| `[À compléter]` | Windows | Sysmon ID 13 | Modification de la Run Key, si enregistrée | Persistance simulée | Événement Sysmon |
| `[À compléter]` | Windows | Fichier local | Création de `stolen-credentials.txt` | Récupération d'identifiants fictifs | Fichier / événement disponible |
| `[À compléter]` | Debian | SSH / auth.log | Authentification de `labops` | Mouvement latéral simulé | Log SSH |
| `[À compléter]` | Debian | auditd | Exécution de `soc-lab-maintenance` | Action privilégiée contrôlée | Événement auditd |
| `[À compléter]` | Debian | FIM | Création de `soc-lab-status.php`, si signalée | Modification du Webroot | Alerte FIM |
| `[À compléter]` | Debian | Apache | Requête HTTP vers le marqueur | Accès au fichier par le client | `access.log` |

Je peux ensuite représenter la chaîne sous cette forme :

```text
Windows : exécution PowerShell
        ↓
Windows : persistance simulée
        ↓
Windows : récupération d'identifiants fictifs
        ↓
Debian : connexion SSH de labops
        ↓
Debian : commande de maintenance en privilèges élevés
        ↓
Debian : création du fichier PHP de test
        ↓
Apache : requête vers le fichier PHP
```

Une chronologie de qualité distingue les faits observés, les hypothèses et les éléments qui n'ont pas pu être confirmés.

---

# 11. Recenser les indicateurs et artefacts du lab

Je prépare une liste des artefacts observés. Je garde uniquement les valeurs qui ont effectivement été confirmées dans mon environnement.

| Type | Valeur à relever | Source de vérification |
|---|---|---|
| Adresse IP | IP de Kali | `hostname -I` sur Kali |
| Adresse IP | IP de Windows | `ipconfig` |
| Adresse IP | IP de Debian | `hostname -I` sur Debian |
| Fichier | `invoice_update.ps1` | Windows / Kali |
| Fichier | `persistence.ps1` | Windows |
| Fichier | `stolen-credentials.txt` | Windows, identifiants fictifs uniquement |
| Registre | `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\OneDriveUpdateLab` | Windows / Sysmon ID 13 |
| Compte | `labops` | Logs SSH et auditd |
| Fichier | `/var/www/html/soc-lab-status.php` | Debian / FIM / Apache |
| Hash | SHA-256 du fichier PHP ou script | `sha256sum` / `Get-FileHash` |

Je ne publie pas le contenu des fichiers contenant les identifiants, même s'ils sont fictifs. Dans le repository GitHub, je masque les mots de passe et je peux remplacer les IP par des valeurs de documentation si leur publication n'est pas nécessaire.

---

# 12. Captures à conserver pour GitHub

Je conserve uniquement les captures qui aident à raconter l'incident :

1. Sysmon Event ID 1 lié à l'exécution PowerShell.
2. Sysmon Event ID 13 lié à la Registry Run Key, si l'événement a été enregistré.
3. Connexion SSH réussie du compte `labops`.
4. Événement auditd correspondant à l'opération de maintenance.
5. Journal de maintenance contenant `caller=labops` et `euid=0`.
6. Événement FIM sur `soc-lab-status.php`, si disponible.
7. Requête Apache vers `soc-lab-status.php`.
8. Timeline remplie avec les heures réellement observées.

Chaque capture doit avoir un nom explicite, par exemple :

```text
screenshots/
├── day5-01-sysmon-process-create.png
├── day5-02-sysmon-registry-runkey.png
├── day5-03-ssh-authentication.png
├── day5-04-auditd-maintenance.png
├── day5-05-fim-webroot.png
├── day5-06-apache-request.png
└── day5-07-incident-timeline.png
```

Avant de les publier, je vérifie qu'elles ne contiennent pas de secrets, de mots de passe, de jetons d'accès ou d'informations personnelles.

---

# 13. Diagnostic si des événements manquent

## 13.1 Sysmon local fonctionne, mais l'événement n'est pas visible dans Wazuh

- Vérifier que l'agent Windows est actif.
- Vérifier la période sélectionnée dans le Dashboard.
- Vérifier que l'événement déclenche une alerte ; si ce n'est pas le cas, il peut être absent de `wazuh-alerts-*`.
- Vérifier les archives si elles sont activées.
- Contrôler les journaux de l'agent et du manager avant de modifier la configuration.

## 13.2 SSH existe dans `auth.log`, mais pas dans les alertes

- Vérifier la présence du bloc `localfile` dans `/var/ossec/etc/ossec.conf` sur Debian.
- Vérifier que `wazuh-agent` est actif.
- Vérifier si l'événement est seulement un événement collecté sans alerte.
- Comparer la période du Dashboard avec l'heure locale du serveur.

## 13.3 auditd ne montre pas la commande de maintenance

- Vérifier que la règle `soc_lab_exec` est chargée avec `sudo auditctl -l`.
- Vérifier que la commande a réellement été exécutée.
- Examiner les événements auditd autour de l'heure du test.
- Ne pas conclure à une élévation de privilèges uniquement sur la base du fichier marqueur : croiser les logs du script avec auditd et sudo.

## 13.4 Le fichier existe mais aucune alerte FIM n'apparaît

- Vérifier que `/var/www/html` figure bien dans la configuration FIM de l'agent Debian.
- Vérifier que l'agent a rechargé sa configuration.
- Vérifier si le fichier a été créé avant ou après le démarrage de la surveillance.
- Consulter les alertes FIM et les journaux de l'agent.

## 13.5 Apache enregistre la requête mais rien n'apparaît dans les alertes

C'est possible : une réponse HTTP normale n'entraîne pas systématiquement une alerte. Je conserve `access.log` comme preuve locale et je vérifie les archives si j'ai besoin de rechercher cet événement brut dans Wazuh.

---

# 14. Résultat attendu à la fin du Jour 5

À la fin de la journée, je dois avoir :

- identifié les traces Sysmon utiles au scénario ;
- recherché la persistance Windows et les fichiers créés ;
- confirmé ou documenté la connexion SSH du compte `labops` ;
- examiné l'opération de maintenance avec auditd ;
- vérifié la création du marqueur dans le Webroot et la requête Apache ;
- distingué les événements présents localement des alertes visibles dans Wazuh ;
- construit une timeline à partir des horodatages réels ;
- recensé les artefacts pertinents pour les Jours 6 et 7.

La timeline n'a pas besoin d'être complète pour être utile. Les éléments manquants doivent être signalés comme des lacunes de collecte ou des points restant à confirmer, au lieu d'être remplacés par des suppositions.

## Conclusion

Pour ce cinquième jour, j'ai commencé l'investigation de la chaîne d'attaque simulée lors des Jours 3 et 4.

J'ai examiné les événements Sysmon sur Windows, les journaux d'authentification SSH sur Debian, les événements auditd, les changements suivis par le FIM et les accès enregistrés par Apache. J'ai ensuite comparé les preuves disponibles sur les machines avec les événements et les alertes visibles dans Wazuh.

Cette étape me permet de comprendre ce que ma télémétrie capture réellement et d'identifier les événements qui pourraient être corrélés pour détecter le scénario. J'ai également documenté les limites de la simulation : les identifiants utilisés sont fictifs et le fichier PHP est un marqueur inoffensif.

Au Jour 6, je vais exploiter les événements réellement observés pour développer et tester des règles Wazuh personnalisées. L'objectif sera de transformer les indices identifiés aujourd'hui en détections reproductibles.

---

# Documentation de référence

- Wazuh — Event logging : https://documentation.wazuh.com/current/user-manual/manager/event-logging.html
- Wazuh — Index des alertes et des archives : https://documentation.wazuh.com/current/user-manual/wazuh-indexer/wazuh-indexer-indices.html
- Wazuh — Auditd : https://documentation.wazuh.com/current/user-manual/capabilities/system-calls-monitoring/audit-configuration.html
- Wazuh — Fonctionnement du FIM : https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/how-it-works.html
- Wazuh — Configuration du FIM : https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/how-to-configure-fim.html
- MITRE ATT&CK — Registry Run Keys / Startup Folder (T1547.001) : https://attack.mitre.org/techniques/T1547/001/
- MITRE ATT&CK — Credentials In Files (T1552.001) : https://attack.mitre.org/techniques/T1552/001/
- MITRE ATT&CK — SSH (T1021.004) : https://attack.mitre.org/techniques/T1021/004/
- MITRE ATT&CK — Sudo and Sudo Caching (T1548.003) : https://attack.mitre.org/techniques/T1548/003/
