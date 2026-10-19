# JOUR 4 - Mouvement latéral SSH et élévation de privilèges sur Debian

## Objectif du jour

Lors du Jour 3, j'ai simulé la compromission d'un poste Windows 11. J'ai utilisé Kali Linux pour héberger un script PowerShell, puis j'ai exécuté ce script sur Windows.

J'ai ensuite simulé une persistance à l'aide d'une Registry Run Key et une récupération d'identifiants fictifs stockés dans un fichier local.

Aujourd'hui, je vais poursuivre ce scénario en utilisant les identifiants de test pour me connecter au serveur Debian depuis le poste Windows.

L'objectif est de reproduire la suite de l'attaque :

```text
WINDOWS 11
    |
    | Identifiants de test récupérés au Jour 3
    v
Connexion SSH vers Debian
    |
    v
Compte labops
    |
    v
Exécution d'une action avec des privilèges root
    |
    v
Modification du répertoire Web Apache
    |
    v
Collecte des traces dans Wazuh
```

Je vais réaliser les opérations dans mon environnement de laboratoire uniquement. Pour l'élévation de privilèges, je vais mettre en place une délégation sudo volontairement trop permissive, mais limitée à une opération précise et inoffensive. Je ne vais pas créer de shell root arbitraire ni de véritable webshell.

À la fin de la journée, je veux être capable de répondre à ces questions :

- Depuis quelle machine la connexion SSH a-t-elle été établie ?
- Quel compte a été utilisé ?
- Quand l'authentification a-t-elle réussi ?
- Quelle opération privilégiée a été exécutée ?
- Quels fichiers ont été créés sur Debian ?
- Quelles traces sont disponibles dans auditd, Apache et Wazuh ?

---

# 1. Vérifier les artefacts du Jour 3

Avant de commencer, je vais vérifier que j'ai bien terminé les étapes du Jour 3.

## 1.1 Vérifier le fichier d'identifiants sur Windows

Sur Windows 11, j'ouvre PowerShell.

Je vérifie que le fichier existe :

```powershell
Get-Item "C:\ProgramData\SOC-Lab\Stage\stolen-credentials.txt"
```

Puis :

```powershell
Get-Content "C:\ProgramData\SOC-Lab\Stage\stolen-credentials.txt"
```

Je dois retrouver les identifiants fictifs que mon script a récupérés lors du Jour 3.

Par exemple :

```text
username=labops
password=Lab-SOC-2026!
retrieved=...
```

Je vérifie également que l'adresse du serveur Debian indiquée dans mon fichier `deployment-notes.txt` correspond bien à l'adresse réelle de mon serveur.

**Important :** le mot de passe ci-dessus est uniquement un exemple de laboratoire. J'utiliserai le mot de passe effectivement configuré pour mon compte `labops`.

## 1.2 Vérifier la persistance Windows

Je vérifie également que la Registry Run Key créée au Jour 3 existe toujours :

```powershell
Get-ItemProperty `
  "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" |
  Select-Object OneDriveUpdateLab
```

Je conserve les artefacts du Jour 3, car ils me serviront pendant les investigations des prochains jours.

---

# 2. Préparer le serveur Debian

Je passe maintenant sur ma machine Debian.

## 2.1 Vérifier l'adresse IP

Je commence par afficher les adresses IP de mon serveur :

```bash
hostname -I
```

Je note l'adresse correspondant à mon réseau de laboratoire.

Je peux également afficher les informations du système :

```bash
hostname
cat /etc/os-release
```

Je dois être certain de travailler sur le bon serveur avant de modifier sa configuration.

## 2.2 Vérifier les services

Je vérifie que Wazuh fonctionne :

```bash
sudo systemctl is-active wazuh-agent
```

Puis auditd :

```bash
sudo systemctl is-active auditd
```

Et Apache :

```bash
sudo systemctl is-active apache2
```

Je m'attends normalement à obtenir :

```text
active
```

pour chacun de ces services.

Si l'un d'eux n'est pas actif, je corrige le problème avant de poursuivre.

## 2.3 Vérifier SSH

Je vérifie si le serveur SSH est déjà présent :

```bash
sudo systemctl status ssh
```

S'il n'est pas installé, je peux l'installer avec :

```bash
sudo apt update
sudo apt install -y openssh-server
```

Puis je l'active :

```bash
sudo systemctl enable --now ssh
```

Je vérifie que SSH écoute sur le port 22 :

```bash
sudo ss -lntp | grep ':22'
```

Je dois retrouver un processus `sshd` qui écoute sur le port 22.

Si SSH est déjà installé et fonctionnel, je ne réinstalle rien.

---

# 3. Préparer le compte `labops`

Pour reproduire le scénario, je vais utiliser le compte de laboratoire indiqué dans mon fichier d'identifiants.

Je commence par vérifier s'il existe déjà :

```bash
id labops
```

Si le compte n'existe pas, je le crée :

```bash
sudo adduser labops
```

Le système va me demander de définir un mot de passe. Je configure le même mot de passe fictif que celui utilisé dans `deployment-notes.txt`.

Je ne dois pas réutiliser ce mot de passe pour un compte réel.

Si le compte existe déjà, je vérifie qu'il utilise bien le mot de passe de laboratoire attendu. Je peux le réinitialiser avec :

```bash
sudo passwd labops
```

Je ne vais pas ajouter `labops` au groupe `sudo`, car le but est précisément d'étudier un compte utilisateur ordinaire qui peut exécuter une opération privilégiée limitée.

Je vérifie enfin :

```bash
id labops
```

Puis :

```bash
getent passwd labops
```

Je dois pouvoir identifier son UID, son groupe principal et son répertoire personnel.

---

# 4. Vérifier la configuration d'auditd

Au Jour 2, j'ai configuré auditd pour surveiller les appels système liés aux exécutions.

Je vais vérifier que la règle de laboratoire est toujours chargée :

```bash
sudo auditctl -l | grep soc_lab_exec
```

Je dois retrouver la clé :

```text
soc_lab_exec
```

Je vérifie également que le journal existe :

```bash
sudo ls -lh /var/log/audit/audit.log
```

Puis :

```bash
sudo tail -n 10 /var/log/audit/audit.log
```

Cette vérification est importante : je veux que les commandes exécutées via SSH puissent laisser des traces exploitables par la suite.

---

# 5. Préparer l'opération privilégiée de laboratoire

## 5.1 Pourquoi utiliser sudo ?

Dans une entreprise, un compte utilisateur peut disposer de droits supplémentaires pour réaliser certaines tâches administratives. Une configuration trop permissive peut permettre à un compte de lancer une opération avec des privilèges supérieurs à ceux qui lui sont normalement accordés.

Dans mon laboratoire, je vais simuler ce risque en autorisant `labops` à exécuter une seule commande de maintenance en tant que root, sans demander de mot de passe.

Cette commande sera volontairement limitée : elle créera un fichier de journalisation et un marqueur PHP inoffensif dans le répertoire Web.

Cela reproduira les traces d'une action privilégiée sans autoriser l'exécution libre de commandes en root.

## 5.2 Vérifier le répertoire Web

Je vérifie que le répertoire Apache existe :

```bash
sudo ls -ld /var/www/html
```

Je conserve ce chemin, car il sera utilisé pour créer le marqueur PHP.

Si ce répertoire n'existe pas, je vérifie la configuration Apache avant de poursuivre.

## 5.3 Créer le script de maintenance

Je crée un script appartenant à root :

```bash
sudo nano /usr/local/sbin/soc-lab-maintenance
```

J'ajoute le contenu suivant :

```bash
#!/bin/bash
set -eu

TIMESTAMP="$(date -Is)"
CALLER="${SUDO_USER:-unknown}"

printf '%s action=soc-lab-maintenance caller=%s euid=%s\n' \
  "$TIMESTAMP" "$CALLER" "$(id -u)" \
  >> /var/log/soc-lab-maintenance.log

cat > /var/www/html/soc-lab-status.php <<'PHP'
<?php
header('Content-Type: text/plain; charset=UTF-8');
echo "SOC-LAB-WEBSHELL-MARKER\n";
?>
PHP

chown root:root /var/www/html/soc-lab-status.php
chmod 644 /var/www/html/soc-lab-status.php

printf '%s\n' \
  "SOC-LAB: maintenance completed"
```

Je sauvegarde le fichier.

Ce script réalise trois opérations :

1. Il écrit une trace dans `/var/log/soc-lab-maintenance.log`.
2. Il crée un fichier PHP dans le Webroot Apache.
3. Il conserve ce fichier sous le propriétaire `root`, avec des permissions permettant à Apache de le lire.

Le fichier PHP est uniquement un marqueur : il n'exécute aucune commande système et n'accepte aucun paramètre permettant de contrôler le serveur.

## 5.4 Protéger le script

Je configure les permissions :

```bash
sudo chown root:root /usr/local/sbin/soc-lab-maintenance
sudo chmod 750 /usr/local/sbin/soc-lab-maintenance
```

Je vérifie :

```bash
sudo ls -l /usr/local/sbin/soc-lab-maintenance
```

Le compte `labops` ne doit pas pouvoir modifier directement ce script.

---

# 6. Configurer la délégation sudo de laboratoire

Je vais maintenant créer une règle sudoers spécifique à mon projet.

J'ouvre un fichier dédié :

```bash
sudo visudo -f /etc/sudoers.d/soc-lab-maintenance
```

J'ajoute uniquement cette ligne :

```text
labops ALL=(root) NOPASSWD: /usr/local/sbin/soc-lab-maintenance
```

Je sauvegarde et je quitte.

Je vérifie ensuite la syntaxe :

```bash
sudo visudo -cf /etc/sudoers.d/soc-lab-maintenance
```

Je dois obtenir une confirmation indiquant que le fichier est syntaxiquement correct.

Je vérifie aussi les permissions :

```bash
sudo chmod 440 /etc/sudoers.d/soc-lab-maintenance
```

Puis :

```bash
sudo ls -l /etc/sudoers.d/soc-lab-maintenance
```

Enfin, je vérifie les droits effectivement accordés à `labops` :

```bash
sudo -l -U labops
```

Je dois constater que ce compte peut exécuter :

```text
/usr/local/sbin/soc-lab-maintenance
```

avec des privilèges root.

**Point important :** cette délégation est volontairement utilisée pour le laboratoire. Elle ne doit pas être copiée telle quelle dans un environnement de production.

---

# 7. Tester la connectivité entre Windows et Debian

Je retourne maintenant sur Windows 11.

Je vérifie que le client SSH est disponible :

```powershell
Get-Command ssh.exe
```

S'il n'est pas disponible, je peux vérifier la fonctionnalité OpenSSH Client dans Windows et l'installer si nécessaire.

Je teste ensuite le port SSH de Debian :

```powershell
Test-NetConnection DEBIAN_IP -Port 22
```

Je remplace `DEBIAN_IP` par l'adresse réelle de mon serveur.

Je recherche :

```text
TcpTestSucceeded : True
```

Si le résultat est `False`, je ne continue pas : je vérifie l'adresse IP, le service SSH, le réseau virtuel et le pare-feu.

---

# 8. Réaliser le mouvement latéral via SSH

Je vais maintenant utiliser les identifiants fictifs récupérés au Jour 3.

Depuis PowerShell sur Windows :

```powershell
ssh labops@DEBIAN_IP
```

Je remplace `DEBIAN_IP` par l'adresse réelle de Debian.

Lors de la première connexion, SSH peut me demander de confirmer l'empreinte de la machine. Je vérifie qu'il s'agit bien de mon serveur de laboratoire avant de l'accepter.

Je saisis ensuite le mot de passe fictif correspondant au compte `labops`.

Si tout fonctionne, je dois arriver sur un terminal Debian.

Je vérifie mon identité :

```bash
whoami
```

Résultat attendu :

```text
labops
```

Je vérifie ensuite :

```bash
id
hostname
date -Is
```

Ces commandes me permettent de confirmer que je suis connecté au bon serveur et sous le bon compte.

À ce stade, le mouvement latéral simulé est réalisé :

```text
Windows 11
    |
    | SSH
    v
Debian
    |
    v
labops
```

Cela correspond à MITRE ATT&CK T1021.004 - SSH.

---

# 9. Exécuter l'opération privilégiée

Je suis maintenant connecté à Debian sous le compte `labops`.

Avant d'exécuter le script, je peux vérifier que je ne suis pas root :

```bash
id -u
```

Résultat attendu :

```text
1001
```

L'UID exact peut varier selon les comptes du système.

Je vérifie que la commande autorisée est accessible :

```bash
sudo -n /usr/local/sbin/soc-lab-maintenance
```

L'option `-n` demande à sudo de ne pas afficher de demande interactive de mot de passe.

Comme cette commande a été autorisée par ma règle de laboratoire, le script devrait s'exécuter sans demande de mot de passe.

Résultat attendu :

```text
SOC-LAB: maintenance completed
```

Cette opération s'exécute avec des privilèges root, mais elle ne transforme pas le terminal interactif de `labops` en shell root.

Je vérifie d'ailleurs :

```bash
whoami
```

Le résultat reste :

```text
labops
```

C'est normal : seule la commande de maintenance autorisée bénéficie des privilèges root.

---

# 10. Vérifier le résultat sur Debian

Toujours depuis le terminal SSH, je vérifie le journal créé par le script :

```bash
sudo tail -n 10 /var/log/soc-lab-maintenance.log
```

Je dois retrouver une ligne contenant notamment :

```text
action=soc-lab-maintenance
caller=labops
euid=0
```

Je vérifie ensuite le fichier PHP :

```bash
ls -l /var/www/html/soc-lab-status.php
```

Puis son contenu :

```bash
cat /var/www/html/soc-lab-status.php
```

Le fichier doit contenir le marqueur :

```text
SOC-LAB-WEBSHELL-MARKER
```

Je vérifie aussi ses permissions :

```bash
stat /var/www/html/soc-lab-status.php
```

Je dispose maintenant de traces démontrant qu'une opération lancée par `labops` a conduit à la création d'un fichier avec des privilèges élevés.

---

# 11. Vérifier les traces SSH

Je peux maintenant quitter ma session SSH :

```bash
exit
```

Sur Debian, je consulte les dernières traces d'authentification.

Si `/var/log/auth.log` est présent :

```bash
sudo grep sshd /var/log/auth.log | tail -n 30
```

Je peux également utiliser le journal systemd :

```bash
sudo journalctl -u ssh --since "30 minutes ago"
```

Je recherche notamment une ligne indiquant une authentification réussie, avec le compte :

```text
labops
```

ainsi que l'adresse IP source de Windows.

Je veux pouvoir établir le lien suivant :

```text
Adresse IP Windows
        |
        v
Connexion SSH
        |
        v
Utilisateur labops
        |
        v
Authentification réussie
```

---

# 12. Vérifier les traces auditd

Je vais maintenant chercher les événements d'exécution de commandes capturés par auditd.

Sur Debian :

```bash
sudo ausearch -k soc_lab_exec -i | tail -n 80
```

Je recherche l'exécution du script :

```text
soc-lab-maintenance
```

ainsi que les informations relatives à l'identité et aux privilèges.

Les champs qui m'intéressent particulièrement sont :

```text
auid
uid
euid
exe
comm
key
```

Je veux notamment pouvoir distinguer l'identité à l'origine de l'action du contexte privilégié dans lequel l'opération a été exécutée.

Dans mon scénario :

```text
Compte à l'origine de l'action : labops
Privilèges de l'opération      : root
```

C'est une distinction importante dans une investigation SOC.

---

# 13. Tester l'accès au fichier PHP via Apache

Je vais maintenant vérifier que le fichier créé est bien servi par Apache.

Depuis Kali Linux, j'exécute :

```bash
curl -i http://DEBIAN_IP/soc-lab-status.php
```

Je remplace `DEBIAN_IP` par l'adresse de mon serveur Debian.

Je dois obtenir une réponse HTTP `200 OK` contenant :

```text
SOC-LAB-WEBSHELL-MARKER
```

Il ne s'agit pas d'un véritable webshell. Ce fichier sert uniquement à reproduire une modification du Webroot et une requête HTTP que je pourrai corréler avec les autres traces.

Je retourne ensuite sur Debian et je consulte le journal Apache :

```bash
sudo tail -n 30 /var/log/apache2/access.log
```

Je recherche :

```text
soc-lab-status.php
```

Je dois retrouver la requête ainsi que l'adresse IP du client qui a effectué la demande.

Je peux alors reconstituer la dernière partie du scénario :

```text
Connexion SSH
      |
      v
Exécution privilégiée
      |
      v
Création d'un fichier PHP
      |
      v
Requête HTTP vers le fichier
      |
      v
Trace dans access.log
```

---

# 14. Vérifier le File Integrity Monitoring dans Wazuh

Lors du Jour 2, j'ai configuré le FIM pour surveiller le répertoire :

```text
/var/www/html
```

Je retourne maintenant dans Wazuh Dashboard et je sélectionne mon agent Debian.

Je recherche les événements de File Integrity Monitoring associés à :

```text
soc-lab-status.php
```

Je souhaite retrouver des informations relatives à la création du fichier, son chemin et, selon les informations remontées, ses permissions et son empreinte.

Je peux ensuite comparer cette trace avec :

- les événements SSH ;
- les événements auditd ;
- le journal Apache.

L'intérêt n'est pas seulement de détecter un fichier créé. C'est de relier ce changement à une séquence d'actions sur le serveur.

**Remarque :** selon la configuration du FIM, la fréquence de collecte et le traitement des événements, l'alerte peut ne pas apparaître instantanément. Si je ne vois rien, je vérifie la configuration du FIM et les journaux de l'agent avant de conclure que la surveillance ne fonctionne pas.

---

# 15. Vérifier les événements côté Windows

Je retourne enfin sur Windows pour examiner les traces de la connexion SSH sortante.

Je recherche les événements Sysmon de création de processus :

```powershell
Get-WinEvent `
  -LogName "Microsoft-Windows-Sysmon/Operational" `
  -MaxEvents 100 |
  Where-Object {
    $_.Id -eq 1 -and $_.Message -match 'ssh.exe'
  } |
  Select-Object -First 10 TimeCreated, Id, Message
```

Si le client OpenSSH a été lancé depuis Windows, je devrais pouvoir retrouver son exécution si elle correspond aux règles de collecte Sysmon actives.

Je peux également rechercher des événements réseau Sysmon ID 3 :

```powershell
Get-WinEvent `
  -LogName "Microsoft-Windows-Sysmon/Operational" `
  -MaxEvents 100 |
  Where-Object {
    $_.Id -eq 3 -and $_.Message -match 'ssh.exe'
  } |
  Select-Object -First 10 TimeCreated, Id, Message
```

La présence d'un événement réseau dépend notamment de la configuration Sysmon appliquée au Jour 1. Son absence ne suffit donc pas à conclure qu'aucune connexion SSH n'a été effectuée.

---

# 16. Corréler les événements dans Wazuh

Je vais maintenant essayer de raconter l'incident à partir des traces collectées.

Je cherche les événements autour de la période pendant laquelle j'ai exécuté les commandes.

La chronologie attendue est la suivante :

| Étape | Source | Élément à rechercher |
|---|---|---|
| Connexion distante | `sshd` / `auth.log` | Connexion réussie du compte `labops` |
| Exécution privilégiée | auditd | Exécution de `soc-lab-maintenance` |
| Modification du Webroot | FIM | Création de `soc-lab-status.php` |
| Accès au fichier | Apache `access.log` | Requête vers `soc-lab-status.php` |
| Activité côté Windows | Sysmon | Création du processus `ssh.exe`, si collectée |

Je peux ainsi relier l'origine de la connexion à l'action privilégiée puis à la modification et à l'accès au serveur Web.

Toutes les traces ne seront pas nécessairement des alertes de sécurité visibles dans la vue par défaut. Certaines seront accessibles dans les événements de l'agent ou dans les journaux locaux : cela fait partie du travail d'investigation.

---

# 17. Captures à conserver pour GitHub

Je vais conserver les preuves les plus intéressantes de cette journée :

1. La session SSH ouverte sous `labops`, sans afficher inutilement le mot de passe.
2. La trace d'authentification réussie dans les journaux Debian.
3. La sortie d'auditd montrant l'exécution de `soc-lab-maintenance`.
4. Le journal `/var/log/soc-lab-maintenance.log` avec `caller=labops` et `euid=0`.
5. Le fichier `soc-lab-status.php` et ses permissions.
6. La requête HTTP correspondante dans `access.log`.
7. L'événement FIM dans Wazuh, s'il est remonté.
8. L'événement Sysmon associé au client SSH, s'il est collecté.

Avant toute publication, je vérifie que les captures ne contiennent aucune adresse publique sensible, aucun identifiant réel et aucun secret provenant d'un autre environnement.

---

# 18. Résultat attendu à la fin du Jour 4

À la fin de cette journée, je dois avoir reproduit la chaîne suivante :

```text
WINDOWS 11
    |
    | Identifiants fictifs préparés au Jour 3
    v
SSH
    |
    v
DEBIAN : labops
    |
    v
Commande de maintenance autorisée par sudo
    |
    v
Exécution de l'opération avec euid=0
    |
    v
Création de soc-lab-status.php
    |
    v
Requête Apache
    |
    v
Traces auditd + SSH + Apache + FIM
    |
    v
Investigation dans Wazuh
```

Je ne supprime pas encore ces artefacts. Ils vont me servir lors des Jours 5 et 6 pour l'investigation, la corrélation et le développement de règles de détection.

## Conclusion

Pour cette quatrième journée, j'ai poursuivi le scénario commencé sur Windows en utilisant des identifiants de laboratoire pour établir une connexion SSH vers Debian.

J'ai ensuite simulé une mauvaise délégation de privilèges au moyen d'une règle sudo limitée à une opération de maintenance. Cette opération a créé un marqueur PHP inoffensif dans le Webroot Apache.

L'objectif était de produire une séquence d'événements que je puisse étudier avec plusieurs sources de télémétrie : authentification SSH, auditd, Apache et FIM.

Je dispose maintenant d'une chaîne d'attaque contrôlée traversant deux systèmes :

```text
Windows
   ↓
Identifiants de test
   ↓
SSH
   ↓
Debian
   ↓
Action privilégiée
   ↓
Modification du Webroot
   ↓
Logs et événements Wazuh
```

La prochaine étape consiste à exploiter ces traces pour mener une investigation structurée : reconstruire la chronologie, identifier les événements importants, comprendre les liens entre les machines et concevoir les détections qui permettront de reconnaître ce scénario.
